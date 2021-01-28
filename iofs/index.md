# io/fs preview?

## FS interface

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

### ReadFile

`ReadFile()` 함수는 매개변수로 받은 경로에서 해당 파일을 읽습니다.

```go
fs.ReadFile(os.DirFS("./"), "a.txt")
```

이렇게 작성하면 해당 경로의 a.txt 파일을 읽습니다.  
읽은 결과는 `[]byte`로 반환됩니다.

### Stat

`Stat()` 함수는 `ReadFile()` 함수와 동일하지만 반환값이 `FileInfo`로 파일 정보를 반환합니다.  
예상하셨을 수도 있지만 `error` 값을 어느정도 공유하기때문에 `os.IsNotExist()` 함수를 이용하여 파일 존재 여부도 확인할 수 있습니다.

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

## 끝

`fs` 패키지가 `ioutil` 패키지를 대체하는 용도로 추가되었다고 생각했는데 알아보니 `os` 패키지에 들어갔습니다.  
그리고 `fs` 패키지는 특정 경로에 대한 작업을 용이하게 하기 위해 만들어졌다고 느꼈습니다.  
간단히 생각해도 모든 폴더를 순회하는 코드를 손으로 짜게 된다면 `fs` 패키지의 `Sub()` 함수가 큰 도움이 될 것같습니다.