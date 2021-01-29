# io/fs and embed in golang 1.16 beta

## FS

`io/fs`에는 `FS`라는 인터페이스를 중심으로 움직입니다.

```go
type FS interface {
  	// Open opens the named file.
  	//
  	// When Open returns an error, it should be of type *PathError
	// with the Op field set to "open", the Path field set to name,
	// and the Err field describing the problem.
	//
	// Open should reject attempts to open names that do not satisfy
	// ValidPath(name), returning a *PathError with Err set to
	// ErrInvalid or ErrNotExist.
	Open(name string) (File, error)
}
```
FS 인터페이스는 `Open(name string) (File, error)` 메서드를 가지면 되는 간단한 인터페이스입니다.

그리고 `fs.File`의 경우 `os.File`의 부분집합이기때문에 다음과같이 `os.Open()`의 결과물을 반환하는 형식으로 구현할 수 있습니다.

```go
type myFS struct {}

func (m myFS) Open(name string) (fs.File, error) {
    return os.Open(name)
}

func main() {
	fmt.Println(fs.ReadDir(myFS{}, "./"))
}
```

이 코드는 현재 경로의 모든 파일을 반환합니다.

---

## os.DirFS

하지만 `os` 패키지엔 `DirFS()`라는 함수가 만들어져있습니다.

```go
func main() {
	fmt.Println(fs.ReadDir(os.DirFS("./"), "."))
}
```

이 코드도 현재 경로의 모든 파일을 반환합니다.

```go
fs.ReadDir(os.DirFS("<prefix>"), "<directory>")
```

이 함수는 prefix와 directory를 위와같이 작성할 수 있습니다.

다른 구문과같이 `/`는 root directory, `.`은 current directory, `..`는 parent directory를 의미합니다.

`DirFS()` 함수의 반환값인 `dirFS`의 `Open()` 메서드를 보면 `f, err := Open(string(dir) + "/" + name)`로 작성되어 directory와 name을 단순 연결하는 걸 알 수 있고 prefix와 directory를 비우지만 않으면 어떤식으로 쓰든 상관없다는 걸 추측할 수 있습니다.

---

## GlobFS, ReadDirFS, ReadDirFile, ReadFileFS, StatFS, SubFS

`fs.GlobFS` 인터페이스는 `FS` 인터페이스에서 `Glob(pattern string) ([]string, error)` 메서드를 추가로 요구합니다.

`fs.ReadDirFS` 인터페이스는 `FS` 인터페이스에서 `ReadDir(name string) ([]DirEntry, error)` 메서드를 추가로 요구합니다.

`fs.ReadFileFS` 인터페이스는 `FS` 인터페이스에서 `ReadFile(name string) ([]byte, error)` 메서드를 추가로 요구합니다.

`fs.StatFS` 인터페이스는 `FS` 인터페이스에서 `Stat(name string) (FileInfo, error)` 메서드를 추가로 요구합니다.

`fs.SubFS` 인터페이스는 `FS` 인터페이스에서 `Sub(dir string) (FS, error)` 메서드를 추가로 요구합니다.

이 인터페이스들의 사용처인 `Glob()`, `ReadDir()`, `ReadFile()`,`Stat()`, `Sub()` 함수들은 해당 추가 요구 메서드가 구현되어 있을 경우 해당 메서드를 실행하고 종료합니다.

하지만 구현되어 있지 않을 경우 `os.DirFS()`를 넘겨줬을 때처럼 해당 경로(혹은 부가 디렉토리)에서 작업하게 됩니다.

아래의 경우는 `os.DirFS()`를 넘겨줬을 때의 행동입니다.

### Glob

`Glob()` 함수는 매개변수로 들어온 경로에서 해당하는 패턴을 검색한 결과를 반환합니다.

```go
fs.Glob(os.DirFS("/"), "*")
```

이런 식으로 작성하게 되면 root directory의 모든 파일을 가져오게됩니다.  
내부적으로 `path.Match()` 함수를 불러와서 사용하므로 해당 함수의 패턴을 그대로 활용할 수 있습니다.  
아래는 `path.Match()` 함수의 패턴들입니다.

```
pattern:
	{ term }
term:
	'*'         matches any sequence of non-/ characters
	'?'         matches any single non-/ character
	'[' [ '^' ] { character-range } ']'
	            character class (must be non-empty)
	c           matches character c (c != '*', '?', '\\', '[')
	'\\' c      matches character c

character-range:
	c           matches character c (c != '\\', '-', ']')
	'\\' c      matches character c
	lo '-' hi   matches character c for lo <= c <= hi
```

### ReadDir

`ReadDir()` 함수는 매개변수로 받은 경로에서 해당 폴더를 읽습니다.

```go
fs.ReadDir(os.DirFS("./"), ".")
```

이렇게 작성하게 되면 현재 경로를 대상 디렉토리로 판단하여 읽습니다.  
해당 함수는 `.`, `..`, `/`같은 표현을 쓸 수 있습니다.

### DirEntry

`ReadDir()` 함수는 `DirEntry` 인터페이스를 반환합니다.

```go
type DirEntry interface {
    Name() string
    IsDir() bool
    Type() FileMode
    Info() (FileInfo, error)
}
```

`DirEntry` 인터페이스는 위 4가지 메서드를 가지고 있습니다.  
`Name()` 메서드는 파일의 이름을 반환합니다.  
`IsDir()` 함수는 파일이 디렉토리인지를 반환합니다.  
`Type()` 함수는 파일의 모드(종류)와 권한 비트 등을 포함하는 `FileMode`를 반환합니다.  
`Info()` 함수는 파일의 정보를 포함하는 `FileInfo` 인터페이스와 `error`를 반환합니다.

### ReadFile

`ReadFile()` 함수는 매개변수로 받은 경로에서 해당 파일을 읽습니다.

```go
fs.ReadFile(os.DirFS("./"), "a.txt")
```

이렇게 작성하면 해당 경로의 a.txt 파일을 읽습니다.  
읽은 결과는 `[]byte`로 반환됩니다.

### Stat

`Stat()` 함수는 `ReadFile()` 함수와 동일하지만 반환값이 `FileInfo` 인터페이스로 파일 정보를 반환합니다.  
예상하셨을 수도 있지만 `error` 값을 어느정도 공유하기때문에 `os.IsNotExist()` 함수를 이용하여 파일 존재 여부도 확인할 수 있습니다.

### FileInfo

`Stat()` 함수가 반환하는 `FileInfo` 인터페이스는 기존 `os.FileInfo`를 대체하는 파일 정보를 보여주는 인터페이스입니다.  
실제로 `os.FileInfo`는 `fs.FileInfo`를 받아서 선언되어 있습니다.

```go
type FileInfo interface {
    Name() string
    Size() int64        // length in bytes for regular files; system-dependent for others
    Mode() FileMode     // file mode bits
    ModTime() time.Time // modification time
    IsDir() bool        // abbreviation for Mode().IsDir()
    Sys() interface{}   // underlying data source (can return nil)
}
```

`Name()` 함수는 파일의 이름을 반환합니다.  
`Size()` 함수는 byte 단위의 파일 크기를 반환합니다.  
`Mode()` 함수는 `FileMode`를 반환합니다.  
`ModTime()` 함수는 최종 수정된 시각을 반환합니다.  
`IsDir()` 함수는 파일이 디렉토리인지를 반환합니다.  
`Sys()` 함수는 해당 파일이 가지는 내부 포맷을 출력합니다.

### Sub

`Sub()` 함수는 매개변수로 받은 경로의 하위 파일 혹은 폴더의 `FS`를 반환합니다.  
예를 들면 해당 경로 내의 `temp` 폴더의 `FS`를 만들고 싶다면 이런 식으로 작성할 수 있습니다.

```go
fs.Sub(os.DirFS("./"), "temp")
```

이렇게 받은 `FS`는 당연히 인터페이스를 충족하고 있기에 다른 함수에도 그대로 쓸 수 있습니다.

---

## os

golang 1.16beta1에서는 기존 `ioutil` 패키지에 있던 `ReadFile()`, `ReadDir()`, `WriteFile()`이 그대로 `os` 패키지로 옮겨졌습니다.  
golang 팀은 굳이 안바꿔도 되도록 할거라고 했지만 새로 작성하는 코드부터는 다음과 같이 작성하시면 될 것입니다.

```go
// 현재 경로의 main.go 파일을 읽어옵니다.
os.ReadFile("./main.go")

// 현재 경로 내 go 폴더를 읽어옵니다.
fmt.Println(os.ReadDir("./go"))

// 현재 경로 내 .tmp 파일에 []byte인 data를 씁니다.
data := []byte{}
os.WriteFile("./.tmp", data)
```

---

## embed

`embed` 패키지는 외부 파일을 golang 코드에 내장시킬 수 있는 지시자(directive)를 제공합니다.  
이 패키지를 사용하려면 `//go:embed`를 필요한 변수 위에 작성하시면 됩니다.

만약 golang 프로젝트 루트 폴더에 helloworld.txt라는 파일에 `hello, world!`라는 텍스트가 작성되어 있다면 아래 코드는 `hello, world!`를 출력합니다.

```go
import (
	_ "embed"
	"fmt"
)

func main() {
	//go:embed helloworld.txt
	var s string
	fmt.Println(s)
}
```

`embed`를 사용하기 위해서는 먼저 `embed` 패키지를 import 해야합니다.  
지금은 패키지 내부 변수나 함수를 사용하지 않기에 _(underscore)를 별칭으로 패키지를 가져와야합니다.  
`s` 변수에는 자동으로 helloworld.txt를 읽어서 나온 `hello, world!`가 저장되게 됩니다.  

```go
import (
	"embed"
	"fmt"
)

func main() {
	//go:embed helloworld.txt
	var s embed.FS
    fmt.Println(s.ReadFile("helloworld.txt"))
    // fmt.Println(fs.ReadFile(s, "helloworld.txt"))
}
```

또한 이런 식으로 `embed.FS`로 받아서 `ReadFile()` 메서드를 호출하여 불러올 수도 있습니다.  
이는 `fs.FS` 인터페이스도 만족시키기에 `fs.ReadFile(s, "helloworld.txt")`같이 `fs.ReadFile()` 함수를 통해서도 불러올 수 있습니다.  
빌드하여 완전히 다른 경로로 옮겨서 테스트 해봤는데 완전히 embed되어 파일이 없어도 자연스럽게 실행되었습니다.

아래 코드는 공식 문서의 코드입니다. 적절한 자료가 없어서 테스트하지 못 했습니다.

```go
package server

import "embed"

//go:embed image/* template/*
//go:embed html/index.html
var content embed.FS
```

이런 식으로 작성하게 되면 image 폴더와 template 폴더, html/index.html을 `content`에 저장합니다.  
아마 `FS` 구조체에 속해 있는 `files *[]file`에 저장하여 해당 file 별로 그에 맞는 행동을 하는 것같습니다.

```go
//go:embed image template html/index.html
var content embed.FS
```

또한 지시자를 짧게 이런식으로도 작성할 수 있습니다.  
이렇게 받은 static file은 `http.FS()` 함수를 이용하여 핸들링할 수 있습니다.

```go
http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(content))))
```

`template.ParseFS()` 함수를 통해 템플릿도 파싱할 수 있습니다.

```go
template.ParseFS(content, "*.tmpl")
```

---

## 끝
