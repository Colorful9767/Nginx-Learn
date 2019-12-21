
default:	build

clean:
	rm -rf Makefile objs

build:
	$(MAKE) -f objs/Makefile

install:
	$(MAKE) -f objs/Makefile install

modules:
	$(MAKE) -f objs/Makefile modules

upgrade:
	/home/admin/Ngnix-Learn/nginx/sbin/nginx -t

	kill -USR2 `cat /var/temp/nginx/run/nginx.pid`
	sleep 1
	test -f /var/temp/nginx/run/nginx.pid.oldbin

	kill -QUIT `cat /var/temp/nginx/run/nginx.pid.oldbin`
