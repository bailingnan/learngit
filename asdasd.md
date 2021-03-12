[TOC]
# 下载所有镜像文件到本地

1. 搭建镜像必须将所有的anaconda官网pkgs全部下载到本地，通过搜索引擎找到了清华大学已经写好的代码(https://github.com/tuna/tunasync-scripts/blob/master/anaconda.py)。使用脚本下载的会和清华大学的anaconda镜像一样，其中的pkgs相对较多，如果不需要的话我们则不必下载，对文件进行修改或直接注释。另外这个脚本下载源是anaconda官网，如果网络不好同样下载会比较慢。

2. 修改清华大学的下载脚本，将下载源更改为清华大学开源镜像。由于清华大学开源镜像未在网页中提供md5，所以对代码进行修改，取消md5检查(`md5_check`全部返回`True`)。详见修改后清华源下载代码。


```python
#!/usr/bin/env python3
import hashlib
import json
import logging
import os
import random
import shutil
import subprocess as sp
import tempfile
from email.utils import parsedate_to_datetime
from pathlib import Path

from pyquery import PyQuery as pq

import requests


DEFAULT_CONDA_REPO_BASE = "https://mirrors.tuna.tsinghua.edu.cn/anaconda"
DEFAULT_CONDA_CLOUD_BASE = "https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud"

CONDA_REPO_BASE_URL = os.getenv("CONDA_REPO_URL", "https://mirrors.tuna.tsinghua.edu.cn/anaconda")
CONDA_CLOUD_BASE_URL = os.getenv("CONDA_COULD_URL", "https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud")

WORKING_DIR = os.getenv("TUNASYNC_WORKING_DIR")

CONDA_REPOS = ("main", "free", "r", "msys2")
# CONDA_ARCHES = (
#     "noarch", "linux-64", "linux-32", "linux-armv6l", "linux-armv7l",
#     "linux-ppc64le", "osx-64", "osx-32", "win-64", "win-32"
# )

CONDA_ARCHES = (
    "linux-64","noarch"
)

# CONDA_CLOUD_REPOS = (
#     "conda-forge/linux-64", "conda-forge/osx-64", "conda-forge/win-64", "conda-forge/noarch",
#     "msys2/linux-64", "msys2/win-64", "msys2/noarch",
#     "rapidsai/linux-64", "rapidsai/noarch",
#     "bioconda/linux-64", "bioconda/osx-64", "bioconda/win-64", "bioconda/noarch",
#     "menpo/linux-64", "menpo/osx-64", "menpo/win-64", "menpo/win-32", "menpo/noarch",
#     "pytorch/linux-64", "pytorch/osx-64", "pytorch/win-64", "pytorch/win-32", "pytorch/noarch",
#     "pytorch-test/linux-64", "pytorch-test/osx-64", "pytorch-test/win-64", "pytorch-test/win-32", "pytorch-test/noarch",
#     "stackless/linux-64", "stackless/win-64", "stackless/win-32", "stackless/linux-32", "stackless/osx-64", "stackless/noarch",
#     "fermi/linux-64", "fermi/osx-64", "fermi/win-64", "fermi/noarch",
#     "fastai/linux-64", "fastai/osx-64", "fastai/win-64", "fastai/noarch",
#     "omnia/linux-64", "omnia/osx-64", "omnia/win-64", "omnia/noarch",
#     "simpleitk/linux-64", "simpleitk/linux-32", "simpleitk/osx-64", "simpleitk/win-64", "simpleitk/win-32", "simpleitk/noarch",
#     "caffe2/linux-64", "caffe2/osx-64", "caffe2/win-64", "caffe2/noarch",
#     "plotly/linux-64", "plotly/linux-32", "plotly/osx-64", "plotly/win-64", "plotly/win-32", "plotly/noarch",
#     "intel/linux-64", "intel/linux-32", "intel/osx-64", "intel/win-64", "intel/win-32", "intel/noarch",
#     "auto/linux-64", "auto/linux-32", "auto/osx-64", "auto/win-64", "auto/win-32", "auto/noarch",
#     "ursky/linux-64", "ursky/osx-64", "ursky/noarch",
#     "matsci/linux-64", "matsci/osx-64", "matsci/win-64", "matsci/noarch",
#     "psi4/linux-64", "psi4/osx-64", "psi4/win-64", "psi4/noarch",
#     "Paddle/linux-64", "Paddle/linux-32", "Paddle/osx-64", "Paddle/win-64", "Paddle/win-32", "Paddle/noarch",
#     "deepmodeling/linux-64", "deepmodeling/noarch",
#     "numba/linux-64", "numba/linux-32", "numba/osx-64", "numba/win-64", "numba/win-32", "numba/noarch",
#     "numba/label/dev/win-64", "numba/label/dev/noarch",
#     "pyviz/linux-64", "pyviz/linux-32", "pyviz/win-64", "pyviz/win-32", "pyviz/osx-64", "pyviz/noarch",
#     "dglteam/linux-64", "dglteam/win-64", "dglteam/osx-64", "dglteam/noarch",
#     "rdkit/linux-64", "rdkit/win-64", "rdkit/osx-64", "rdkit/noarch",
#     "mordred-descriptor/linux-64", "mordred-descriptor/win-64", "mordred-descriptor/win-32", "mordred-descriptor/osx-64", "mordred-descriptor/noarch",
#     "ohmeta/linux-64", "ohmeta/osx-64", "ohmeta/noarch",
#     "qiime2/linux-64", "qiime2/osx-64", "qiime2/noarch",
#     "biobakery/linux-64", "biobakery/osx-64", "biobakery/noarch",
#     "c4aarch64/linux-aarch64", "c4aarch64/noarch",
#     "pytorch3d/linux-64", "pytorch3d/noarch",
#     "idaholab/linux-64", "idaholab/noarch",
# )
CONDA_CLOUD_REPOS = (
    "pytorch/linux-64", "pytorch/noarch",
    "conda-forge/linux-64", "conda-forge/noarch",
    "msys2/linux-64", "msys2/noarch",
    "rapidsai/linux-64", "rapidsai/noarch",
    "bioconda/linux-64",  "bioconda/noarch",
    "menpo/linux-64", "menpo/noarch",
    "pytorch-test/linux-64",  "pytorch-test/noarch",
    "stackless/linux-64",  "stackless/noarch",
    "fermi/linux-64", "fermi/noarch",
    "fastai/linux-64", "fastai/noarch",
    "omnia/linux-64", "omnia/noarch",
    "simpleitk/linux-64",  "simpleitk/noarch",
    "caffe2/linux-64", "caffe2/noarch",
    "plotly/linux-64", "plotly/noarch",
    "intel/linux-64", "intel/noarch",
    "auto/linux-64",  "auto/noarch",
    "ursky/linux-64", "ursky/noarch",
    "matsci/linux-64",  "matsci/noarch",
    "psi4/linux-64", "psi4/noarch",
    "Paddle/linux-64", "Paddle/noarch",
    "deepmodeling/linux-64", "deepmodeling/noarch",
    "numba/linux-64", "numba/noarch",
    "numba/label/dev/win-64", "numba/label/dev/noarch",
    "pyviz/linux-64", "pyviz/noarch",
    "dglteam/linux-64", "dglteam/noarch",
    "rdkit/linux-64","rdkit/noarch",
    "mordred-descriptor/linux-64", "mordred-descriptor/noarch",
    "ohmeta/linux-64", "ohmeta/noarch",
    "qiime2/linux-64", "qiime2/noarch",
    "biobakery/linux-64", "biobakery/noarch",
    "c4aarch64/linux-aarch64", "c4aarch64/noarch",
    "pytorch3d/linux-64", "pytorch3d/noarch",
    "idaholab/linux-64", "idaholab/noarch",
)

EXCLUDED_PACKAGES = (
    "pytorch-nightly", "pytorch-nightly-cpu", "ignite-nightly",
)

# connect and read timeout value
TIMEOUT_OPTION = (7, 10)

logging.basicConfig(
    level=logging.INFO,
    format="[%(asctime)s] [%(levelname)s] %(message)s",
)

def sizeof_fmt(num, suffix='iB'):
    for unit in ['','K','M','G','T','P','E','Z']:
        if abs(num) < 1024.0:
            return "%3.2f%s%s" % (num, unit, suffix)
        num /= 1024.0
    return "%.2f%s%s" % (num, 'Y', suffix)

def md5_check(file: Path, md5: str = None):
    # m = hashlib.md5()
    # with file.open('rb') as f:
    #     while True:
    #         buf = f.read(1*1024*1024)
    #         if not buf:
    #             break
    #         m.update(buf)
    # return m.hexdigest() == md5
    return True


def curl_download(remote_url: str, dst_file: Path, md5: str = None):
    sp.check_call([
        "curl", "-o", str(dst_file),
        "-sL", "--remote-time", "--show-error",
        "--fail", "--retry", "10", "--speed-time", "15",
        "--speed-limit", "5000", remote_url,
    ])
    if md5 and (not md5_check(dst_file, md5)):
        return "MD5 mismatch"


def sync_repo(repo_url: str, local_dir: Path, tmpdir: Path, delete: bool):
    logging.info("Start syncing {}".format(repo_url))
    local_dir.mkdir(parents=True, exist_ok=True)

    repodata_url = repo_url + '/repodata.json'
    bz2_repodata_url = repo_url + '/repodata.json.bz2'
    # https://docs.conda.io/projects/conda-build/en/latest/release-notes.html
    # "current_repodata.json" - like repodata.json, but only has the newest version of each file
    current_repodata_url = repo_url + '/current_repodata.json'

    tmp_repodata = tmpdir / "repodata.json"
    tmp_bz2_repodata = tmpdir / "repodata.json.bz2"
    tmp_current_repodata = tmpdir / 'current_repodata.json'

    curl_download(repodata_url, tmp_repodata)
    curl_download(bz2_repodata_url, tmp_bz2_repodata)
    try:
        curl_download(current_repodata_url, tmp_current_repodata)
    except:
        pass

    with tmp_repodata.open() as f:
        repodata = json.load(f)

    remote_filelist = []
    total_size = 0
    packages = repodata['packages']
    if 'packages.conda' in repodata:
        packages.update(repodata['packages.conda'])
    for filename, meta in packages.items():
        if meta['name'] in EXCLUDED_PACKAGES:
            continue

        file_size, md5 = meta['size'], meta['md5']
        total_size += file_size

        pkg_url = '/'.join([repo_url, filename])
        dst_file = local_dir / filename
        dst_file_wip = local_dir / ('.downloading.' + filename)
        remote_filelist.append(dst_file)

        if dst_file.is_file():
            stat = dst_file.stat()
            local_filesize = stat.st_size

            if file_size == local_filesize:
                logging.info("Skipping {}".format(filename))
                continue

            dst_file.unlink()

        for retry in range(3):
            logging.info("Downloading {}".format(filename))
            try:
                err = curl_download(pkg_url, dst_file_wip, md5=md5)
                if err is None:
                    dst_file_wip.rename(dst_file)
            except sp.CalledProcessError:
                err = 'CalledProcessError'
            if err is None:
                break
            logging.error("Failed to download {}: {}".format(filename, err))


    shutil.move(str(tmp_repodata), str(local_dir / "repodata.json"))
    shutil.move(str(tmp_bz2_repodata), str(local_dir / "repodata.json.bz2"))
    if tmp_current_repodata.is_file():
        shutil.move(str(tmp_current_repodata), str(
            local_dir / "current_repodata.json"))

    if delete:
        local_filelist = []
        delete_count = 0
        for i in local_dir.glob('*.tar.bz2'):
            local_filelist.append(i)
        for i in local_dir.glob('*.conda'):
            local_filelist.append(i)
        for i in set(local_filelist) - set(remote_filelist):
            logging.info("Deleting {}".format(i))
            i.unlink()
            delete_count += 1
        logging.info("{} files deleted".format(delete_count))

    logging.info("{}: {} files, {} in total".format(
        repodata_url, len(remote_filelist), sizeof_fmt(total_size)))
    return total_size

def sync_installer(repo_url, local_dir: Path):
    logging.info("Start syncing {}".format(repo_url))
    local_dir.mkdir(parents=True, exist_ok=True)
    full_scan = random.random() < 0.1 # Do full version check less frequently

    def remote_list():
        r = requests.get(repo_url, timeout=TIMEOUT_OPTION)
        d = pq(r.content)
        for tr in d('table').find('tr'):
            tds = pq(tr).find('td')
            if len(tds) != 4:
                continue
            fname = tds[0].find('a').text
            md5 = tds[3].text
            yield (fname, md5)

    for filename, md5 in remote_list():
        pkg_url = "/".join([repo_url, filename])
        dst_file = local_dir / filename
        dst_file_wip = local_dir / ('.downloading.' + filename)

        if dst_file.is_file():
            r = requests.head(pkg_url, allow_redirects=True, timeout=TIMEOUT_OPTION)
            len_avail = 'content-length' in r.headers
            if len_avail:
                remote_filesize = int(r.headers['content-length'])
            remote_date = parsedate_to_datetime(r.headers['last-modified'])
            stat = dst_file.stat()
            local_filesize = stat.st_size
            local_mtime = stat.st_mtime

            # Do content verification on ~5% of files (see issue #25)
            if (not len_avail or remote_filesize == local_filesize) and remote_date.timestamp() == local_mtime and \
                    (random.random() < 0.95 or md5_check(dst_file, md5)):
                logging.info("Skipping {}".format(filename))

                # Stop the scanning if the most recent version is present
                # if not full_scan:
                #     logging.info("Stop the scanning")
                #     break

                continue

            logging.info("Removing {}".format(filename))
            dst_file.unlink()

        for retry in range(3):
            logging.info("Downloading {}".format(filename))
            err = ''
            try:
                err = curl_download(pkg_url, dst_file_wip, md5=md5)
                if err is None:
                    dst_file_wip.rename(dst_file)
            except sp.CalledProcessError:
                err = 'CalledProcessError'
            if err is None:
                break
            logging.error("Failed to download {}: {}".format(filename, err))

def main():
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--working-dir", default=WORKING_DIR)
    parser.add_argument("--delete", action='store_true',
                        help='delete unreferenced package files')
    args = parser.parse_args()

    if args.working_dir is None:
        raise Exception("Working Directory is None")

    working_dir = Path(args.working_dir)
    size_statistics = 0
    random.seed()

    logging.info("Syncing installers...")
    # for dist in ("archive", "miniconda"):
    #     remote_url = "{}/{}".format(CONDA_REPO_BASE_URL, dist)
    #     local_dir = working_dir / dist
    #     try:
    #         sync_installer(remote_url, local_dir)
    #         size_statistics += sum(
    #             f.stat().st_size for f in local_dir.glob('*') if f.is_file())
    #     except Exception:
    #         logging.exception("Failed to sync installers of {}".format(dist))

    for repo in CONDA_REPOS:
        for arch in CONDA_ARCHES:
            remote_url = "{}/pkgs/{}/{}".format(CONDA_REPO_BASE_URL, repo, arch)
            local_dir = working_dir / "pkgs" / repo / arch

            tmpdir = tempfile.mkdtemp()
            try:
                size_statistics += sync_repo(remote_url,
                                             local_dir, Path(tmpdir), args.delete)
            except Exception:
                logging.exception("Failed to sync repo: {}/{}".format(repo, arch))
            finally:
                shutil.rmtree(tmpdir)

    for repo in CONDA_CLOUD_REPOS:
        remote_url = "{}/{}".format(CONDA_CLOUD_BASE_URL, repo)
        local_dir = working_dir / "cloud" / repo

        tmpdir = tempfile.mkdtemp()
        try:
            size_statistics += sync_repo(remote_url,
                                         local_dir, Path(tmpdir), args.delete)
        except Exception:
            logging.exception("Failed to sync repo: {}".format(repo))
        finally:
            shutil.rmtree(tmpdir)

    print("Total size is", sizeof_fmt(size_statistics, suffix=""))

if __name__ == "__main__":
    main()

# vim: ts=4 sw=4 sts=4 expandtab
```

## 脚本使用

运行脚本前需取得外网访问权限,如`curl`命令需要配置代理。

```python
python anaconda.py --working-dir=/your/download/path
```

下载目录注意要选择在挂载盘，不要在系统盘，否则容量远远不够，例如我的下载目录为`/root/mnt/mirror/anaconda`。

## 建立索引
下载的文件会在 pkgs 根目录下，在下载目录中我们需要运行以下命令
```shell
conda index pkgs/*
```

# 搭建conda服务
为了使局域网内的用户都可访问本地Anaconda镜像，我们首先搭建一个本地http服务器

- 在Ubuntu中通过`apt-get install apache2` 安装apache2

  CentOS中通过`yum install httpd` 安装httpd

- apache2的配置文件是`/etc/apache2/apache2.conf`

  httpd的配置文件是`/etc/httpd/conf/httpd.conf`

- 配置文件两者默认的访问端口都是80端口，这是可以在配置文件中进行修改的。

- 在配置文件中可以发现，服务器默认的访问路径在`/var/www/html`目录下。只是为了简单实现一台http文件服务器，因此可以在此目录下创建一个软连接来连接文件目录。

  创建软链接，例如我们的镜像 pkgs 文件夹在 `/root/mnt/mirror/anaconda/pkgs`，在 `/var/www/html` 目录下通过命令 `ln -s /root/mnt/mirror/anaconda /var/www/html` 创建一个软连接。

# 本地conda使用配置
使用`vi /root/.condarc `修改配置文件，

和清华大学开源镜像的帮助指南一样，将其修改为:

```
channels:
  - http://服务器地址:监听端口/anaconda/pkgs/main
  - http://服务器地址:监听端口/anaconda/pkgs/r
  - http://服务器地址:监听端口/anaconda/pkgs/msys2
show_channel_urls: true
ssl_verify: false
custom_channels:
  conda-forge: http://服务器地址:监听端口/anaconda/cloud
  msys2: http://服务器地址:监听端口/anaconda/cloud
  bioconda: http://服务器地址:监听端口/anaconda/cloud
  menpo: http://服务器地址:监听端口/anaconda/cloud
  pytorch: http://服务器地址:监听端口/anaconda/cloud
  simpleitk: http://服务器地址:监听端口/anaconda/cloud
```
# 更新源

需要重复上面的镜像下载步骤即可。

# 定时更新源
为了使得镜像及时更新，我们可以使用 Linux 的 crontab 服务定时更新 `anaconda.py` 脚本。具体方法如下：

运行 `crontab -e` 编写一条定时任务：

```shell
0 1  * * 6 /usr/bin/python /root/mnt/mirror/anaconda.py --working-dir /root/mnt/mirror/anaconda > /root/mnt/mirror/auto.log
```

意思是每周六的凌晨 1:00 执行 `anaconda.py` 脚本。

运行 `crontab -l` 进行验证。

# 参考链接

- [清华大学开源软件镜像站Anaconda主页](https://mirrors.tuna.tsinghua.edu.cn/anaconda/)
- [清华大学开源软件镜像站Anaconda 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/)
- [搭建本地Anaconda镜像](https://xungejiang.com/2019/07/06/local-anaconda-mirror/)
- [手把手教你搭建Anaconda镜像源](https://cloud.tencent.com/developer/article/1618504)
- [conda镜像源安装说明](http://code-cbu.huawei.com/ModelArts/Train/docs/blob/master/ci/DLS-CI.md#conda%E9%95%9C%E5%83%8F%E6%BA%90%E5%AE%89%E8%A3%85%E8%AF%B4%E6%98%8E)


