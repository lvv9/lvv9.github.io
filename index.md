visit my:

- [tumblr](http://tumblr.liuweiqiang.me/)
- [douban](https://www.douban.com/people/liriban/)

my posts:

{% for post in site.posts %}
- {{ post.date | date_to_string }} [{{ post.title }}]({{ post.url }}){% endfor %}
