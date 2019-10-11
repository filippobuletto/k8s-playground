# Windows Extra

## Use scoop to install the tools

Install [scoop](https://github.com/lukesampson/scoop#installation) and execute the following commands:

```powershell
# Must
scoop install git-with-openssh
scoop install kubectl
scoop install vagrant
# Extra buckets
scoop bucket add extras
scoop bucket add Ash258 'https://github.com/Ash258/scoop-Ash258.git'
scoop bucket add pot-pourri 'https://github.com/filippobuletto/pot-pourri'
scoop update
# Nice to have
scoop install kubebox
scoop install kubefwd
scoop install kubeval
scoop install octant
scoop install rakkess
scoop install stern
scoop install popeye
scoop install mkcert
```
