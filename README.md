# blog
Made with [jekyll](http://jekyllrb.com) and hosted with Github Pages.


### Usage

* fork this repository
* edit the `_config.yml` with your info
* change the links in `_data/navigation.yml`
* remove my posts from `_posts/`
* rename your repo to ***your-username*.github.io**

### License
All this stuff is under the [MIT License](https://raw.githubusercontent.com/getmicah/getmicah.github.io/master/LICENSE)

### Test Locally with Docker Desktop

```sh
docker run -it --rm -p 127.0.0.1:4000:4000 --platform=linux/amd64 -v $PWD:/srv/jekyll -w /srv/jekyll --entrypoint=sh ruby:2.5.0 -c 'bundle install && bundle exec jekyll serve --host=0.0.0.0'
```
