# Sail Review

sail log 분석을 위한 웹 어플리케이션이다. 사용자가 ULog sail logs를 업로드하고 분석할 수 있다.

[bokeh](http://bokeh.pydata.org) library를 사용하여 차트를 그리고 [Tornado Web Server](http://www.tornadoweb.org)를 사용한다.

![Plot View](screenshots/plot_view.png)

## 3D View
![3D View](screenshots/3d_view.gif)


## 설치 및 설정

### 요구사항

- Python3 (3.6+ recommended)
- SQLite3
- [http://fftw.org/](http://fftw.org/)

#### Ubuntu

```bash
sudo apt-get install sqlite3 fftw3 libfftw3-dev
```

**Note:** Under some Ubuntu and Debian environments you might have to
install ATLAS

```bash
sudo apt-get install libatlas3-base
```

### 설치

```bash
# After git clone, enter the directory
git clone --recursive https://github.com/BadaProject/sail_review.git
cd sail_review/app
pip install -r requirements.txt
# Note: preferably use a virtualenv

# Note: if you get an error about "ModuleNotFoundError: No module named 'libevents_parse'" update submodules
git submodule update --init --recursive
```

### Setup

DB 초기화 :

```bash
./app/setup_db.py
```

**Note:** `setup_db.py` 으로 DB table을 업그레이드하는데 사용할 수 있다. 예를 들면 새로운 entries가 추가되는 경우(자동으로 검출).

#### 설정

- 기본적으로 app은 `config_default.ini` 설정 파일을 로드한다.
- ...
- By default the app will load `config_default.ini` configuration file
- You can override any setting from `config_default.ini` with a user config file
  `config_user.ini` (untracked)
- Any setting on `config_user.ini` has priority over
  `config_default.ini`

## 사용법

For local usage, the server can be started directly with a log file name,
without having to upload it first:

```bash
cd app
./serve.py -f <file.ulg>
```

To start the whole web application:
```bash
cd app
./serve.py --show
```

The `plot_app` directory contains a bokeh server application for plotting. It
can be run stand-alone with `bokeh serve --show plot_app` (or with `cd plot_app;
bokeh serve --show main.py`, to start without the html template).

The whole web application is run with the `serve.py` script. Run `./serve.py -h`
for further details.

## Interactive Usage
The plotting can also be used interative using a Jupyter Notebook. It
requires python knowledge, but provides full control over what and how to plot
with immediate feedback.

- Start the notebook
- Locate and open the test notebook file `testing_notebook.ipynb`.

```bash
# Launch jupyter notebook
cd app
jupyter notebook testing_notebook.ipynb
```

# Implementation
The web site is structured around a bokeh application in `app/plot_app`
(`app/plot_app/configured_plots.py` contains all the configured plots). This
application also handles the statistics page, as it contains bokeh plots as
well. The other pages (upload, browse, ...) are implemented as tornado handlers
in `app/tornado_handlers/`.

`plot_app/helper.py` additionally contains a list of log topics that the plot
application can subscribe to. A topic must live in this list in order to be
plotted.

Tornado uses a single-threaded event loop. This means all operations should be
non-blocking (see also http://www.tornadoweb.org/en/stable/guide/async.html).
(This is currently not the case for sending emails).

Reading ULog files is expensive and thus should be avoided if not really
necessary. There are two mechanisms helping with that:
- Loaded ULog files are kept in RAM using an LRU cache with configurable size
  (when using the helper method). This works from different requests and
  sessions and from all source contexts.
- There's a LogsGenerated DB table, which contains extracted data from ULog
  for faster access.

## Caching
In addition to in-memory caching there is also some on-disk caching: KML files
are stored on disk. Also the parameters and airframes are cached and downloaded
every 24 hours. It is safe to delete these files (but not the cache directory).

## Notes about python imports
Bokeh uses dynamic code loading and the `plot_app/main.py` gets loaded on each
session (page load) to isolate requests. This also means we cannot use relative
imports. We have to use `sys.path.append` to include modules in `plot_app` from
the root directory (Eg `tornado_handlers.py`). Then to make sure the same module
is only loaded once, we use `import xy` instead of `import plot_app.xy`.
It's useful to look at `print('\n'.join(sys.modules.keys()))` to check this.

# Docker usage

docker로 작업하는 방법을 알아보자.

## Arguments

설정을 위해서 `.env` 파일을 수정한다.

- PORT - docker에서 service listen을 위한 port number. 기본값은 5006
- USE_PROXY - The set his, if you use reverse proxy (Nginx, ...)
- DOMAIN - The address domain name for origin, default = *
- CERT_PATH - The SSL certificate volume path
- EMAIL - Email for challenging Let's Encrypt DNS

## Paths

- /opt/service/config_user.ini - Path for config
- /opt/service/data - Folder where stored database
- .env - Environment variables for nginx and app docker container

## Build Docker Image

```bash
cd app
docker build -t px4flightreview -f Dockerfile .
```

## docker-compose로 작업하기
docker container를 구동시키기 위해서 아래 명령을 실행한다.
`.env`를 수정하고 `app/config_user.ini`를 추가한다.

개발할때는 로컬 IP 주소로 `BOKEH_ALLOW_WS_ORIGIN` 부분을 코멘트를 해제한다. 이렇게 해야
bokeh application의 websocket이 동작한다.

### Development
```bash
docker-compose -f docker-compose.dev.yml up
```

### Test Locally
nginx로 로컬 테스트:

```bash
docker-compose up
```

`default_ssl.conf`를 사용하기 위해서 `NGINX_CONF`를 변경하는 것을 기억하고, `EMAIL`을 추가한다.

### Production
```bash
htpasswd -c ./nginx/.htpasswd username
# here to create a .htpasswd for nginx basic authentication
chmod u+x init-letsencrypt.sh
./init-letsencrypt.sh
```