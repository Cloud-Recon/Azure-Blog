<!DOCTYPE html>
<html lang="{{ site.lang | default: "en-US" }}">
  <head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ page.title | default: site.title }}</title>
    <link rel="stylesheet" href="{{ '/assets/css/style.css?v=' | append: site.github.build_revision | relative_url }}">
  </head>

  <body>
    <header class="page-header" role="banner">
      <h1 class="project-name">{{ site.title }}</h1>
      <h2 class="project-tagline">{{ site.description }}</h2>

      <p>
        <a href="https://www.linkedin.com/in/wayne-marks" target="_blank" rel="noopener">LinkedIn: Wayne Marks</a><br>
        <a href="https://cloud-recon.github.io/Azure-Blog/">Take me home</a>
      </p>
    </header>

    <main id="content" class="main-content" role="main">
      {{ content }}
      <footer class="site-footer">
        <p>
          © {{ site.time | date: "%Y" }} {{ site.title }}. All rights reserved.
        </p>
      </footer>
    </main>
  </body>
</html>
