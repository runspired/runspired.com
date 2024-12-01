```ts
type Author = {
    name: 'Chris Thoburn';
    alias: '@runspired';
    title: 'Software Engineer';
    description: 'OpenSource Contributor to Data Frameworks';
    topics: 'WarpDrive' | 'EmberData' | 'EdgePipes' | 'MPAs' | 'SPAs';
    socials: {
      bluesky: "@runspired.com";
      github: "runspired";
    }
};
```

## Posts

<ul>
  {% for post in site.posts %}
    {% unless post.draft %}
    <li>
      <time datetime="{{ post.date | date_to_xmlschema }}"></time>{{ post.date | date: "%Y-%m-%d" }}
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
    {% endunless %}
  {% endfor %}
</ul>
