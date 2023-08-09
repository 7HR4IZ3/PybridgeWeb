# PybridgeWeb
Modified version of pybridge for use mainly with Js Bridge Web

## Example

Async using starlette (asgi framework), domonic (html builder), aiohttp and aiohttp_asgi (web server)

```python
import asyncio
import datetime
import time
from domonic import html as dom

from starlette.routing import Router
from starlette.requests import Request
from starlette.websockets import WebSocket
from starlette.background import BackgroundTask
from starlette.responses import PlainTextResponse, Response, HTMLResponse, JSONResponse

from pybridge import AsyncMultiServer, BridgeJS, daemon_task, force_sync, async_daemon_task

app = Router()
wrserver = AsyncMultiServer()

app.route("/__web_route_js__")(lambda *a: PlainTextResponse(BridgeJS))

@app.websocket_route("/__web_route_ws__/{conn_id:str}")
async def _(websocket: WebSocket):
    await websocket.accept()

    # handle_connection: Main handler for pybridge
    await wrserver.handle_connection(websocket, websocket.path_params['conn_id'])

@app.route("/")
async def index(request: Request):
    return PlainTextResponse("My Index Page!")

@app.route("/counter")
async def counter(request: Request):
    
    # new_connection: Tells wrserver to expect a connextion
    # conn_id refers to the connection id
    # script refers to the javascript script to be injects in the response which initializes the js client
    conn_id, script = wrserver.new_connection()

    @async_daemon_task
    async def background():
        
        # get_connection: 
        browser = await wrserver.get_connection(conn_id)

        select = lambda *a: browser.document.querySelector(*a)

        counts = await select("#counts")
        name_el = await select("#name")
        increment_btn = await select("#incrementBtn")

        async def increment(event):
            counts.innerText = int(await counts.innerText) + 1
            # await browser.alert("Hello")

        async def sayName(event):
            name = await browser.prompt("Name: ")
            name_el.innerText = f"Hello '{name}'"

        await increment_btn.addEventListener("click", increment)
        await (await select("#nameBtn")).addEventListener("click", sayName)

    return HTMLResponse(
        script + str(dom.div(
            dom.span("Name:&nbsp;", dom.p(id="name")),
            dom.span("Count:&nbsp;", dom.p(0, id="counts")),
            
            dom.button("Increment", id="incrementBtn"),
            dom.button("Enter name", id="nameBtn")
        )), background=BackgroundTask(background)
    )

@app.route("/clock")
def clock(request: Request):
    conn_id, script = wrserver.new_connection()

    @BackgroundTask
    @async_daemon_task
    async def background():
        browser = await wrserver.get_connection(conn_id)

        select = lambda *a: browser.document.querySelector(*a)
        timeEl = await select("#time")

        while True:
            timeEl.innerText = datetime.now().ctime()
            await asyncio.sleep(1)

    return HTMLResponse(script + str(
        dom.div("Time: ", dom.p(id="time"))
    ), background=background)

if __name__ == "__main__":
    from aiohttp import web
    from aiohttp_asgi import ASGIResource

    aiohttp_app = web.Application()
    asgi_resource = ASGIResource(app, root_path="/")
    aiohttp_app.router.register_resource(asgi_resource)
    asgi_resource.lifespan_mount(aiohttp_app)
    web.run_app(aiohttp_app)
```
