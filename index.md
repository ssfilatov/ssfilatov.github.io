## Async Python in Openstack Horizon

##### Brief intro
Python 3.4 introduced brand new async library - **asyncio** and that's awesome!

* Python's GIL doesn't let us use benefits of multicore processor - threads are not as effective as they could be.

* Switching context consumes more resources in threads than in coroutines.

* In some cases asynchronous code will improve your application vastly.

If you are new to async features in python - here's good article [How the heck does async await work in python3.5](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/)

This page contains thoughts on making [Openstack Horizon](https://github.com/openstack/horizon) faster by making it work with async requests.

### Sync Horizon Imitation

For testing purposes I took code responsible for rendering **instances** page:

```python
def get_data(self):
    instances = []
    marker = self.request.GET.get(
        project_tables.InstancesTable._meta.pagination_param, None)

    search_opts = self.get_filters({'marker': marker, 'paginate': True})

    # Gather our flavors and images and correlate our instances to them
    try:
        flavors = api.nova.flavor_list(self.request)
    except Exception:
        flavors = []
        exceptions.handle(self.request, ignore=True)

    try:
        # TODO(gabriel): Handle pagination.
        images = api.glance.image_list_detailed(self.request)[0]
    except Exception:
        images = []
        exceptions.handle(self.request, ignore=True)

    if 'image_name' in search_opts and \
            not swap_filter(images, search_opts, 'image_name', 'image'):
            self._more = False
            return instances
    elif 'flavor_name' in search_opts and \
            not swap_filter(flavors, search_opts, 'flavor_name', 'flavor'):
            self._more = False
            return instances

    # Gather our instances
    try:
        instances, self._more = api.nova.server_list(
            self.request,
            search_opts=search_opts)
    except Exception:
        self._more = False
        instances = []
        exceptions.handle(self.request,
                          _('Unable to retrieve instances.'))
```                              

I wanted to make sync and async code more or less similar(and Openstack Horizon does not support python 3), so I rewrote and simplified(**a lot!**) *get_data* function:

```python
def build_novaclient(auth_token):
    return nova_client.Client('2.1',
                           USERNAME,
                           project_name=PROJECT_NAME,
                           project_domain_id=PROJECT_DOMAIN_ID,
                           auth_url=AUTH_URL,
                           auth_token=auth_token)

def build_glanceclient(auth_token):
    return glance_client.Client('2', GLANCE_URL, token=auth_token)

def build_neutronclient(auth_token):
    return neutron_client.Client(token=auth_token, 
        endpoint_url = NEUTRON_URL, auth_url = AUTH_URL)


def get_auth_token():
    auth = v3.Password(auth_url=AUTH_URL, username=USERNAME,
                    password=PASSWORD, project_name=PROJECT_NAME,
                    user_domain_id=USER_DOMAIN_ID, 
                    project_domain_id=PROJECT_DOMAIN_ID)
    sess = session.Session(auth=auth)
    return sess.get_token()

def get_data():
    auth_token = get_auth_token()

    flavors = build_novaclient(auth_token).flavors.list()
    flavors = [f.name for f in flavors]

    images_iter = build_glanceclient(auth_token).images.list()
    images = list(images_iter)
    images = [i.name for i in images]

    servers = build_novaclient(auth_token).servers.list()
    servers = [s.name for s in servers]

    ports = build_neutronclient(auth_token).list_ports()
    networks = build_neutronclient(auth_token).list_networks()
    fips = build_neutronclient(auth_token).list_floatingips()

    return servers, images, flavors
```

### Async Horizon Imitation

First, we need to set App object from **aiohttp** lib:
```python
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
app = web.Application()
app.router.add_route("GET", "/", get_data)
web.run_app(app, port=8080)
```

To make coroutines work, we need to create asynchronous http requests. Openstack clients that we used above do not work for us. I used **aiohttp.ClientSession** lib:

```python
async def list_servers(auth_token):

    url = "NOVA_URL%s" % ("/servers/detail")
    headers = {'content-type': 'application/json', 
        'X-Auth-Token': auth_token}
    async with ClientSession() as session:
        async with session.get(url, headers=headers) as response:
            response = await response.json()
            return [s['name'] for s in response['servers']]
```

We need to attach our couroutines to the main loop and gather them all in our routing function:

```python
async def get_data(request):
    auth_token = get_auth_token()
    res_list = await asyncio.gather(
        list_flavors(auth_token),
        list_servers(auth_token),
        list_images(auth_token),
        list_networks(auth_token),
        list_ports(auth_token),
        list_fips(auth_token),
    )
    response = web.Response(body="\n".join(
        [str(x) for x in res_list]).encode())
    return response
```

### Benchmarks

Permormed on empty non-HA mode OpenStack locally

ApacheBenchmarks:

| Concurrency level   |      Sync code     |  Async code   |  Sanic code |
|---------------------|:------------------:|--------------:|------------:|
|          5          |      559.749ms     |    142.543ms  |  150.435ms  |
|          20         |          *         |    144.334ms  |  136.909ms  |
|          100        |          *         |    141.432ms  |  140.570ms  |


*My terrible code crushes with concurrency level more than 5*

### ToDO

Performed over vpn on stacked HA mode OpenStack:

Average time in sync mode:

**4.86 sec** per request

Average time in async mode:

**2.15 sec** per request

