all: hello

hello: hello.c
	gcc hello.c -o hello

install:
	install -m 755 hello /usr/local/bin/

clean:
	rm -f hello