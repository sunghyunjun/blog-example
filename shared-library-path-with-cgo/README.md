<!-- {%comment%} -->
# Shared Library Path 이해와 cgo 사용하기
<!-- {%endcomment%} -->

Shared Library를 cgo와 함께 사용하는데 있어 도움이 되는 정보를 기록합니다.

## LD_LIBRARY_PATH

LD_LIBRARY_PATH는 각종 library를 사용할 때 참조 path 문제를 해결하고자 사용하곤 합니다.

```bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/shared/lib
```

위와 같은 식으로 사용이 가능합니다. 하지만 LD_LIBRARY_PATH 쉘 환경변수 설정은 디버깅, 개발 시에만 제한적으로 사용되는 것이 바람직합니다.

## Why LD_LIBRARY_PATH is Bad

위의 주제로 구글링을 해보면 많은 글들을 찾아볼 수 있습니다. 그 중 일부를 살펴보면 아래와 같습니다.

[Why LD_LIBRARY_PATH is bad](http://xahlee.info/UnixResource_dir/_/ldpath.html)

[When should I set LD_LIBRARY_PATH?](http://linuxmafia.com/faq/Admin/ld-lib-path.html)

> LD_LIBRARY_PATH is used in preference to any run time or default system linker path. If (God forbid) you had it set to something like /dcs/spod/baduser/lib, if there was a hacked version of libc in that directory (for example) your account could be compromised. It is for this reason that set-uid programs completely ignore LD_LIBRARY_PATH.

> When code is compiled and depends on this to work, it can cause confusion where different versions of a library are installed in different directories, for example there is a libtiff in /usr/openwin/lib and /usr/local/lib. In this case, the former library is an older one used by some programs that come with Solaris.

LD_LIBRARY_PATH를 쉘 환경변수로 global 하게 설정할 경우, 이후 실행되는 모든 애플리케이션에서 shared library를 참조하는 과정에 있어 보안위험, 잘못된 버전 파일 링크 위험이 있다는 것입니다.

실제 서비스, 프로덕트를 배포하는 데 있어 docker 등으로 환경을 제한한다면 이런 위험성은 줄어들지만, 이런 지식을 모든 개발자, 사용자가 알고 있는 것은 아니기 때문에 애당초 이런 위험성을 노출시키지 않도록 작업을 해놓는 것이 좋다고 봅니다.

## -Wl,-rpath,path/to/lib

[Gurus say that LD_LIBRARY_PATH is bad - what's the alternative?](https://stackoverflow.com/questions/882110/gurus-say-that-ld-library-path-is-bad-whats-the-alternative)

ld 옵션을 살펴보면 위와 같은 옵션이 있습니다. RPATH, RUNPATH 에 대한 부분입니다.

이것을 활용하면 링크 후 실행파일에서 링크된 shared library를 어느 path에서 찾아야 하는지를 전달할 수 있습니다.

rpath 관련하여 가장 잘 되어 있는 설명은 아래와 같습니다.

[What's the difference between \`-rpath-link\` and \`-L\`?](https://stackoverflow.com/questions/49138195/whats-the-difference-between-rpath-link-and-l)

아래의 글들도 도움이 됩니다.

[Better understanding Linux secondary dependencies solving with examples](http://www.kaizou.org/2015/01/linux-libraries.html)

[I don't understand -Wl,-rpath -Wl,](https://stackoverflow.com/questions/6562403/i-dont-understand-wl-rpath-wl)

## Linux ld manpage

좀 더 자세하고 확실한 정보는 아래에서 확인할 수 있습니다.

아래의 링크들에서 rpath로 검색을 해보면 됩니다.

[Ubuntu Manpage: ld](https://manpages.ubuntu.com/manpages/xenial/man1/ld.1.html)

> [-rpath=dir](https://manpages.ubuntu.com/manpages/xenial/man1/ld.1.html#:~:text=s%0A%20%20%20%20%20%20%20%20%20%20%20and%20%2DS.-,%2Drpath%3Ddir,-Add%20a%20directory)

[ld.so(8) — Linux manual page](https://man7.org/linux/man-pages/man8/ld.so.8.html)

[ld-linux(8) - Linux man page](https://linux.die.net/man/8/ld-linux)

## cgo 에서의 활용

[cgo - Cgo enables the creation of Go packages that call C code](https://pkg.go.dev/cmd/cgo)

```cpp
#cgo CXXFLAGS: -I${SRCDIR}/path/to/lib
#cgo LDFLAGS: -L${SRCDIR}/path/to/lib -Wl,-rpath=${SRCDIR}/path/to/lib -lmyshared
```

cgo를 사용하여 go에서 C++ shared library를 사용할 때 위와 같은 식으로 rpath 사용이 가능합니다. ${SRCDIR}은 cgo에서 지원되는 지시어입니다.

> When the cgo directives are parsed, any occurrence of the string ${SRCDIR} will be replaced by the absolute path to the directory containing the source file. This allows pre-compiled static libraries to be included in the package directory and linked properly. For example if package foo is in the directory /go/src/foo:

여기서 추가로 아래의 옵션도 사용을 고려할 수 있습니다.

`-Wl,-rpath=\$ORIGIN`

`-Wl,--disable-new-dtags`

`-Wl,--enable-new-dtags`

이 옵션들은 하나로 합쳐서 아래와 같이 사용이 가능합니다.
`-Wl,-rpath=/path/to/lib,--disable-new-dtags`

이에 대한 설명은 앞서 소개한 linux man 페이지에서 확인할 수 있습니다.

아래의 글도 참고 하실 수 있습니다.

[How to set RPATH and RUNPATH with GCC/LD?](https://stackoverflow.com/questions/52018092/how-to-set-rpath-and-runpath-with-gcc-ld)

[ld: Using -rpath,$ORIGIN inside a shared library (recursive)](https://stackoverflow.com/questions/6323603/ld-using-rpath-origin-inside-a-shared-library-recursive)

## 특수한 상황

위와 같은 rpath 설정을 해도 path 관련 not found 오류가 발생하는 경우가 있습니다.

shared library 간에 secondary dependency가 있는 경우에 각 library ELF 헤더에 RUNPATH가 지정된 경우입니다.

secondary dependancy 가 있는 구조여도 RPATH를 `--disable-new-dtags`와 함께 설정하면 모든 라이브러리에 대한 RPATH가 지정되어야 하지만 예외가 있습니다.

[Ubuntu Manpage: ld](https://manpages.ubuntu.com/manpages/xenial/man1/ld.1.html)

> [-rpath-link=dir](https://manpages.ubuntu.com/manpages/xenial/man1/ld.1.html#:~:text=6.%20%20For%20a%20native%20ELF%20linker%2C%20the%20directories%20in%20%22DT_RUNPATH%22%20or%20%22DT_RPATH%22%20of%20a%20shared%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20library%20are%20searched%20for%20shared%20libraries%20needed%20by%20it.%20The%20%22DT_RPATH%22%20entries%20are%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20ignored%20if%20%22DT_RUNPATH%22%20entries%20exist.)

> 6.  For a native ELF linker, the directories in "DT_RUNPATH" or "DT_RPATH" of a shared library are searched for shared libraries needed by it. The "DT_RPATH" entries are ignored if "DT_RUNPATH" entries exist.

기존 라이브러리가 컴파일 되면서 이미 ELF 헤더에 RUNPATH가 기록되어 있다면 RPATH를 실행파일 링크시에 적용해도 RPATH는 무시가 됩니다.

이럴 경우에는 해당 라이브러리의 ELF 헤더의 RUNPATH, RPATH 정보를 직접 수정해줘야 합니다.

[Can I change 'rpath' in an already compiled binary?](https://stackoverflow.com/questions/13769141/can-i-change-rpath-in-an-already-compiled-binary)

```bash
patchelf --set-rpath '$ORIGIN' <shared-lib>
```

ELF 헤더를 확인하기 위해서는 아래와 같이 합니다.

```bash
readelf -d <path-to-shared-lib> |grep PATH
```
