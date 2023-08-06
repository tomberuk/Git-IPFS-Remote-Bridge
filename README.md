# Git IPFS Remote Bridge
**Git IPFS Remote Bridge** is the set of programs written in Python 3 which allow Git user to clone, push, fetch, self-host or release Git repositories over [IPFS](https://ipfs.tech) decentralized data storage system.

## Overview
Git IPFS Remote Bridge is written in Python 3. It provides the following programs: 
- `git-ipfs` - the user interface program intended to be invoked by Git via wrapper program with`git ipfs` abbreviated command. This program act like a frontend solution allowing the user to install/remove IPFS remote, maintain the settings dedicated to communication with IPFS node. Also it provides the instrument to prepare a [**release snapshot**](#Release) from the given tag or commit, and immediately publish it separately on IPFS network.
- `git-remote-ipfs` - the remote helper program. It normally should not be invoked by the end-user directly. Git invokes it maintaining the IPFS node address used as remote URL, which has format like: `url = ipfs://<node-id>`. The program addresses `push`, `fetch`, and `list` commands of Git protocol to IPFS HTTP API of local node (by default) or remote node located somewhere in the network. The Git repository published on IPFS network is an immutable entry, so in the simplest case the program just calls `git remote set-url` to set the new [CID](https://docs.ipfs.tech/concepts/content-addressing/) as remote URL after pushing the data to IPFS. Otherwise, the [IPNS](https://docs.ipfs.tech/concepts/ipns/#how-ipns-works) cryptographically-signed entry key could be specified as remote URL. In this case, the program automatically invokes IPFS node API to automatically associate the obtained immutable CID calculated from the pushed repository data, with specified IPNS key.

Algorithms and logic here are partially inspired by [Dropbox Git helper](https://github.com/anishathalye/git-remote-dropbox) project.

## Features
- **Git repository maintenance**. All the basic Git operations for repository maintenance and remote communication are supported.
- **Self-hosted Git data.**. In case the  IPFS node on which the `push` operation is performed, is connected to the global IPFS network, the repository does not require anymore the server with installed Git for being accessible from anywhere. Moreover, the CID associated with the given state of the repository may be *pinned* being once known, to any IPFS node on the network improving the redundancy and reliability of the stored data.
- **Backward-compatibility**. The pushed repository data available on IPFS network are prepared also for URI-based dumb HTTP cloning. The updated repository would be afterwards shared directly as immutable IPFS entry via HTTP/HTTPS IPFS gateway. The ordinary `git clone` command will work with this repository as usual.
- **IPNS security and immutability**. If the IPNS entry is specified as a remote, the IPFS node will allow associating it with the repository only if the **private key** associated with this entry, is available on its host machine. Thus, the IPNS updates of any kind, made using the programs, are out-of-the-box protected and cryptographically signed. Only the key owher may release the data using his private key, so there are plenty of opportunities for data provenance.

### Releases
The very special feature of the `git-ipfs` program is a possibility to order Git to prepare a **release snapshot** from the given commit or (usually) annotated tag. Executing `git ipfs release` command, the program performs the following steps:
1. Orders Git via invocation of the `git-archive` program to prepare an archive containing snapshot of the repository state addressed by specified commit ID or tag name.
2. Upload the created archive to IPFS network node obtaining an immutable entry, path-like IPFS link via prefix (`/release.tar.gz` by default) and associated CID. 
3. If IPNS public key is specified on the CLI, it will be associated with the obtained CID. Thus, the release snapshot will be published under IPNS entry key with all the protection features described above.

## Limitations
- The program is **not compatible with Git LFS!**. In case both [Git LFS](https://git-lfs.com/) and Git IPFS Bridge hooks are installed, the LFS marked data **will not be stored on IPFS network**! Only the internal links will be packaged and stored on IPFS instead of the actual files, the marked data will be still uploaded on LFS server if its address was configured properly before.
- The upload process is **single-threaded**. Due to HTTP API implementation used, the upload process utilizes only one HTTP connection at the moment. This limitation affects uploading of the large repositories and requires sometimes adjustment of the `Timeout` parameter accordingly.
- The program does not follow IPFS directory entry upload specification, maintaining only path-based addressing of the repository objects on the remote side. This was done to stay compatible with old Git clients supporting only `text/plain`-encoded HTTP responses during dumb cloning.

## Installation
The installation scripts for the program are WIP. Now it is enough to copy the program executables somewhere to the directory mentioned in `PATH`. For example, on Debian Linux this could be done as following:
```sh
sudo cp -v git-ipfs /usr/local/bin
sudo cp -v git-remote-ipfs /usr/local/bin
sudo chmod +x -v /usr/local/bin/git-ipfs /usr/local/bin/git-remote-ipfs
```

### Prerequisites
Git IPFS Bridge uses `requests`, `urllib`, `pathlib`, and `json` modules. From them only `requests` is the third-party module out of the standard library. Please refer to your OS module installation system documentation to learn how to install the required modules and make them available for Python interpreter associated with Git.

To make the programs working, the functioning IPFS node with HTTP API entry point is needed. Please refer to https://docs.ipfs.tech to learn how to install **Kubo** or another implementation of IPFS node on your system.

## Usage
For simplest case, the usage of the program starts from the following command invoked from the rood directory of the Git repository:
```sh
git ipfs install <URL>
```
where `<URL>` is an API URL addressing HTTP endpoint of the chosen IPFS node (for the local node likely `http://127.0.0.1`). This command will create IPFS infrastructure scripts and register them as Git hooks, then initialize a baremetal repository on IPFS as an entry point for IPFS data serializer. Afterwards, the `origin` remote address table will be created on Git and the obtained baremetal repository CID will be associated with the created remote. In case the `origin` remote was already exist before invocation of the command.

The repository with installed IPFS infrastructure can be used as any other Git repository. But when the `push` operation will be executed, Git will upload all data to IPFS and then tune up the remote links to comply with immutable CID changes. In case it is needed to uninstall IPFS infrastructure, it can be done with command:
```sh
git ipfs uninstall
```
All the previously defined remote settings, include URLs which were existing before installation of the IPFS infrastructure, will be restored.

### Cloning
The program supports direct cloning of the repository from IPFS network. It can be done with command:
```sh
git ipfs clone <ID> <directory> <URL>
```
where:
- `<ID>` is IPFS CID or IPNS entry key to clone the repository from. It will be also used as the remote address.
- `<directory>` is a relative path to the directory in which the downloaded repository data will be placed.
- `<URL>` is an IPFS API URL as it was described above.

### Commands and online help
The program works with the commands formulated like the following:
```sh
git ipfs <command> <mode> [options] <arguments>
```
Every command has its own online help. The `-h` key should be used to view the online help of the given command. In the help message, all available modes and options for the specified command will be described. Note that if the argument/option is **not required**, its name will be written within the brackets (`[]`). For example, for `clone` command the online help looks as the following:
```
git ipfs clone -h
usage: git-ipfs clone [-h] [-t TIMEOUT] [-r [REMOTE_NAME]] [-b [BRANCH]] [-n [USERNAME]] [-p [PASSWORD]] ipfs_id directory api_url [api_port]

positional arguments:
  ipfs_id               IPFS CID or IPNS peer name to use as remote ID
  directory             Relative directory to clone the repository in
  api_url               IPFS node API URL (API must be active to view the remote Git database!), default is http://127.0.0.1
  api_port              IPFS node API port (will be attached to URL) [5001]

options:
  -h, --help            show this help message and exit
  -t TIMEOUT, --timeout TIMEOUT
                        Network timeout for API communications, sec (float)
  -r [REMOTE_NAME], --remote-name [REMOTE_NAME]
                        Gives the remote name to make an IPFS remote, default is origin
  -b [BRANCH], --branch [BRANCH]
                        Gives name of the branch to checkout
  -n [USERNAME], --username [USERNAME]
                        HTTP authentication username
  -p [PASSWORD], --password [PASSWORD]
                        HTTP authentication password
```

Every working mode also has its own online help. If the user specifies `-h` key after mentioning the working mode, for example, `git ipfs config manage -h`, the online help for this mode will be printed.

