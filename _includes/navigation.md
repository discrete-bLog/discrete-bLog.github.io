<nav>

{% for item in site.data.navigation %}

  <a href="{{ item.link }}" {% if page.url == item.link %}style="color: cyan;"{% endif %}>
    {{ item.name}}
  </a>
{% endfor %}

</nav>
