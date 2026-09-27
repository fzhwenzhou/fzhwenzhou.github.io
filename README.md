# Zihao Fang’s academic website

A multi-page Jekyll website for GitHub Pages, using the original Minimal Mistakes theme and its existing styling, with the original personal blog preserved.

## Edit content

- `index.html`: biography and academic background.
- `_pages/research.md`: research directions and selected projects.
- `_pages/publications.md`: intentionally blank; add publications here when ready.
- `_pages/cv.md`: education, experience, teaching, and skills.
- `_pages/contact.md`: contact page.
- `_config.yml`: site metadata, portrait path, email addresses, and profile links.
- `_data/navigation.yml`: main navigation.

The supplied portrait is `assets/images/zihao_fang.png`. Pages use the existing `single` layout; the blog uses the existing `home` layout. Theme styles in `assets/css/main.scss` and `_sass/` are unchanged.

There are no resume or CV download links. The resume PDF has been removed. `Resume.pdf`, `resume.pdf`, and the local `CV_English.pdf` draft are excluded from the Jekyll build. When the CV is ready for publication, remove its exclusion and add a link explicitly.

## Personal blog

The archive lives at `/blog/`. All original Markdown files in `_posts/` and their existing URLs are retained. To add a post, create `_posts/YYYY-MM-DD-title.md` with a title and optional categories and `toc: true` in YAML front matter. The archive lists posts in reverse chronological order using the original theme’s post list. Category and tag archives use the theme’s existing layouts.

The shared disclaimer in `_includes/blog-disclaimer.html` appears on both the archive and every post. Older autobiographical content is retained as historical writing, with its original publication date.

## Preview locally

Use a current Ruby installation and Bundler:

```sh
bundle install
bundle exec jekyll serve
```

Open `http://localhost:4000`. For a production build:

```sh
JEKYLL_ENV=production bundle exec jekyll build
```

The site uses the repository’s existing Jekyll setup and GitHub Pages URL. Generated output in `_site/` should not be committed. No Node build step is needed.

## Content references

Current education, academic email, projects, teaching, experience, and skills are based on the previously supplied resume; the PDF itself is not published. Current research direction and interests are based on the [GitHub profile README](https://github.com/fzhwenzhou/fzhwenzhou/blob/master/README.md). Earlier research supervision and HPC team participation are described in the original 2024 introduction. LinkedIn is linked as supplied; its profile content was not accessible during the update.

The site retains the original Minimal Mistakes layouts, colors, typography, navigation, and responsive behavior. The only addition to the post layout is the shared personal-blog disclaimer, using the theme’s built-in notice style.
