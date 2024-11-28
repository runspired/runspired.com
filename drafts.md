```ts
type Author = {
    name: 'Chris Thoburn';
    alias: '@runspired';
    title: 'Software Engineer';
    description: 'OpenSource Contributor to Data Frameworks';
    topics: 'WarpDrive' | 'EmberData' | 'EdgeData' | 'ServerSideData';
};
```

## Drafts

<ul>
  {% for post in site.posts %}
    {% if post.draft %}
    <li>
      <time datetime="{{ post.date | date_to_xmlschema }}"></time>{{ post.date | date: "%Y-%m-%d" }}
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
    {% endif %}
  {% endfor %}
</ul>