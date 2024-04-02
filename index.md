Here you can say lots of fun things about your site.

Maybe say a some things about yourself.

Or maybe what you plan to blog about.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>