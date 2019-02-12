---
layout: post
title: Version in Golang
published: true
---

I am writing more Go services enough to just forgot the current version in production/staging. With nodejs, it is as simple as exporting from package.json.

Go does not have a thing like package.json, and I would expect the `version` should return more than just the version: build time, git hash, and git tag. That information will be inserted into the binary during the build.

At first, the needed information from the shell:

```{shell}
- timestamp: `date -u '+%Y-%m-%d_%I:%M:%S%p'`. (e.g. 2018-10-13_08:26:34AM)  
- githash: `git rev-parse HEAD`
- gittag: `git describe --tags $(git rev-list --tags --max-count=1)`
```

How to pass the value to the artifact? From one issue in Go <https://github.com/golang/go/issues/18246>, yay Go should need a better doc for this, the method is
`-ldflags "-X importpath.name=value"`.

Then the makefile:

```{makefile}
buildtime=`date -u '+%Y-%m-%d_%I:%M:%S%p'`
githash=`git rev-parse HEAD`
gittag=`git describe --tags $(git rev-list --tags --max-count=1)`
LDFLAGS="-X gitlab.com/nqd/example/handlers.buildstamp=$buildtime -X gitlab.com/nqd/example/handlers.githash=$githash -X gitlab.com/nqd/example/handlers.gittag=$gittag"  
```

The makefile tells Go compiler to replace buildstamp, githash, and gittag from value pass by the shell variables. Build the artifact with this LDFLAGS: `$(GO) build -ldflags $(LDFLAGS) ...`

At handlers.go where the buildstamp, githash, and gittag can be exported via one API:

```{golang}
var buildstamp = "no buildstamp provided"  
var githash = "no githash provided"  
var gittag = "no tag"  
  
type statuser struct {
	Build string `json:"build"`  
	Time string `json:"time"`  
	Version string `json:"version"`  
}  
  
func (s *statuser) Render(w http.ResponseWriter, r *http.Request) error {  
	render.Status(r, 200)  
	return nil  
}  
  
func status(w http.ResponseWriter, r *http.Request) {  
	st := statuser{  
		Build: githash,  
		Time: buildstamp,  
		Version: gittag,  
	}  
	render.Render(w, r, &st)  
}  
```

For the same method, we can inject any information into binary artifact during compiling.