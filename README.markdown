Demoserver
==========

Demoserver is a simple git-powered webserver. It allows you to throw quick static demos up using git and let other people clone them.

To use it, just git clone it, `npm install`, set up your config.json, and run `npm start`.

When you want to push a new demo, just `git add remote demoserver https://user:password@yourserver/demoname`. The demo will then be available at https://yourserver/demoname and other people can clone it with `git clone https://yourserver/demoname`.