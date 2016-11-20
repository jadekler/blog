---
authors:
- jadekler
categories:
- Testing
- Go
- Agouti
- Chrome
- UI Testing
- UI
date: 2016-11-17T13:04:17-07:00
draft: true
short: |
  Acceptance tests for your UI are an excellent way to cover user functionality. Let's see how to write a simple acceptance test in Go with Agouti and have it run headlessly in a CI environment with Chrome.
title: Headless UI Testing with Go, Agouti, and Chrome
---

## Foreword
In this blog post we'll be using the following:

- [Go](https://golang.org/) - testing language
- [Ginkgo](https://github.com/onsi/ginkgo) and [gomega](https://github.com/onsi/gomega) - test framework and matchers
- [Agouti](https://github.com/sclevine/agouti) - our webdriver client and UI testing library
- [ChromeDriver](https://sites.google.com/a/chromium.org/chromedriver/) - webdriver
- [xvfb](https://www.x.org/archive/X11R7.6/doc/man/man1/Xvfb.1.xhtml) - a virtual framebuffer X server to allow chrome to run headlessly
- [concourse.ci](https://concourse.ci/) - CI server, whose tasks run on [Ubuntu](https://www.ubuntu.com/) [Docker](https://www.docker.com/) containers

**Note:** All code is assumed to live at `$GOPATH/src/website` (feel free to change `website` to anything)

**Note:** This post assumes you have a running concourse installation

## Writing a simple web app

Let's create a simple `main.go`:

```go
package main

import "log"
import "net/http"

func main() {
  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("<html><body>Hello World</body></html>"))
  })
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

We can now `go run main.go` and navigate to [localhost:8080](localhost:8080) and see our UI!

## Writing a simple test

First, let's get our dependencies:

```bash
go get github.com/onsi/ginkgo/ginkgo
go get github.com/onsi/gomega
go get github.com/sclevine/agouti
```

Then, let's set up our suite - `website_suite_test.go`:

```go
package main_test

import (
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"

	"github.com/onsi/gomega/gexec"
	"github.com/sclevine/agouti"
	"os"
	"os/exec"
	"testing"
)

var (
	agoutiDriver   *agouti.WebDriver
	page           *agouti.Page
	websiteSession *gexec.Session
)

func TestWebsite(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Website Suite")
}

var _ = BeforeSuite(func() {
	agoutiDriver = agouti.ChromeDriver()
	Expect(agoutiDriver.Start()).To(Succeed())

	startWebdriver()
	startWebsite()
})

var _ = AfterSuite(func() {
	Expect(page.Destroy()).To(Succeed())
	Expect(agoutiDriver.Stop()).To(Succeed())
	websiteSession.Kill()
})

func startWebdriver() {
	var err error
	if os.Getenv("TEST_ENV") == "CI" {
		page, err = agouti.NewPage("http://127.0.0.1:9515",
			agouti.Desired(agouti.Capabilities{
				"chromeOptions": map[string][]string{
					"args": []string{
						"disable-gpu", // There is no GPU on our Ubuntu box!
						"no-sandbox", // Sandbox requires namespace permissions that we don't have on a container
					},
				},
			}),
		)
	} else {
		page, err = agoutiDriver.NewPage(agouti.Browser("chrome"))
	}
	Expect(err).NotTo(HaveOccurred())
}

func startWebsite() {
	command := exec.Command("go", "run", "main.go")
	Eventually(func() error {
		var err error
		websiteSession, err = gexec.Start(command, GinkgoWriter, GinkgoWriter)
		return err
	}).Should(Succeed())
}
```

And finally our test - `website_test.go`:

```go
package main_test

import (
	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	. "github.com/sclevine/agouti/matchers"
)

var _ = Describe("Website", func() {
	It("Displays hello world", func() {
		Expect(page.Navigate("http://127.0.0.1:8080/")).To(Succeed())
		Eventually(page).Should(HaveURL("http://127.0.0.1:8080/"))
		Expect(page.Find("body")).To(HaveText("Hello World"))
	})
})
```

Run `ginkgo` to see this work.

## Setting up your CI environment

ConcourseCI uses Docker containers to run tasks, so let's create a container that can run our test. Create the following `Dockerfile`:

```
FROM ubuntu:14.04

ENV LANG="C.UTF-8"

# install utilities
RUN apt-get update
RUN apt-get -y install wget --fix-missing
RUN apt-get -y install xvfb --fix-missing # chrome will use this to run headlessly
RUN apt-get -y install unzip --fix-missing

# install git
RUN apt-get -y install git --fix-missing
RUN apt-get -y install git-core --fix-missing

# install go
RUN wget -O - 'https://storage.googleapis.com/golang/go1.7.linux-amd64.tar.gz' | tar xz -C /usr/local/
ENV PATH="$PATH:/usr/local/go/bin"

# install dbus - chromedriver needs this to talk to google-chrome
RUN apt-get -y install dbus --fix-missing
RUN apt-get -y install dbus-x11 --fix-missing
RUN ln -s /bin/dbus-daemon /usr/bin/dbus-daemon     # /etc/init.d/dbus has the wrong location
RUN ln -s /bin/dbus-uuidgen /usr/bin/dbus-uuidgen   # /etc/init.d/dbus has the wrong location

# install chrome
RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add -
RUN sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list'
RUN apt-get update
RUN apt-get -y install google-chrome-stable
RUN wget -N http://chromedriver.storage.googleapis.com/2.25/chromedriver_linux64.zip
RUN unzip chromedriver_linux64.zip
RUN chmod +x chromedriver
RUN mv -f chromedriver /usr/local/bin/chromedriver
```

Let's build our `Dockerfile`: `docker build . -t my-ci-container`

## Running your test headlessly

Let's now enter the container and run test scripts as if we were the concourse task:

1. First, enter the container with your entire $GOPATH mounted:

    ```
    docker run -it -v $(echo $GOPATH/src/website):/gopath my-ci-container
    ```
1. Let's run the following in the container:

    ```
    export GOPATH=/gopath
    export PATH=$PATH:$GOPATH/bin
    go get github.com/onsi/ginkgo/ginkgo
    go get github.com/onsi/gomega
    go get github.com/sclevine/agouti
    cd $GOPATH/src/website
    xvfb-run chromedriver &
    TEST_ENV=CI ginkgo -v
    ```