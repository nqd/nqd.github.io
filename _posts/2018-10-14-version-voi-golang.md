---
layout: post
title: Version với Golang
published: true
---

# version với golang  
  
Dạo này tôi hay viết dịch vụ với golang, chỉ một vài cái thôi, nhưng cũng đủ để làm tôi quên mất là production/staging đang chạy version nào.  Mục tiêu là thông qua API nào đấy trả về 3 thông tin: thời gian build, git hash, và git tag.
  
Với nodejs chẳng hạn, thì tôi sẽ export version từ package.json ra một api nào đấy.  
  
Với golang thì không có thứ tựa như package.json. Và thật sự cũng không cần thiết, vì những thông tin này có thể đưa vào binary file khi build.  
  
Những thông tin tôi cần quan tâm được lấy trực tiếp từ shell  
- timestamp: `date -u '+%Y-%m-%d_%I:%M:%S%p' `
(e.g. 2018-10-13_08:26:34AM)  
- githash: `git rev-parse HEAD `
- gittag: `git describe --tags $(git rev-list --tags --max-count=1)  `
  
Vấn đề tiếp theo là làm thế nào để đưa các thông tin này vào lúc tạo artifact? Từ một issue của golang https://github.com/golang/go/issues/18246, (yay, golang cần doc tốt hơn) cách thức là  thêm
  
`-ldflags "-X importpath.name=value"  `
  
Cuối cùng, cờ của tôi sẽ có giá trị như sau  
  
```
buildtime=`date -u '+%Y-%m-%d_%I:%M:%S%p'`
githash=`git rev-parse HEAD`
gittag=`git describe --tags $(git rev-list --tags --max-count=1)`
LDFLAGS="-X gitlab.com/nqd/example/handlers.buildstamp=$buildtime -X gitlab.com/nqd/example/handlers.githash=$githash -X gitlab.com/nqd/example/handlers.gittag=$gittag"  
```

Cờ này bảo rằng tôi sẽ thay thế biến buildstamp, githash, và gittag bằng những giá trị từ shell.  
  
Dịch project với cờ thêm vào này  
  
`$(GO) build -ldflags $(LDFLAGS) ...  `
  
Ở handlers.go, nơi chứa các biến buildstamp, githash, gittag, có thể xuất ra một path nào đấy:  
  
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

Cùng một cách thức tương tự tôi có thể đưa bất cứ thông tin cố định nào vào trong binary trong lúc dịch.