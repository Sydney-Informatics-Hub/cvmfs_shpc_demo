# template-mkdocs

## Setup

- Create new repository using this as a template

- In the cloned repository: Settings -> Pages -> Build and deployment:

    - Source: Deploy from a branch
    - Branch: `gh-pages`
    - Folder: `/ (root)`

- Set up `mkdocs` locally

Using `mamba`:

```bash
mamba create -n mkdocs -c conda-forge mkdocs-material
mamba activate mkdocs
```

```console
.
├── docs # all contents for pages here
│   ├── assets
│   │   └── usyd-logo.png
│   ├── extra.css # usyd styling
│   └── index.md # home page
├── .github # GH action to auto publish when pushed to main
│   └── workflows
│       └── mkdocs_deploy.yml
├── mkdocs.yml # config for extensions, contents/navbar, etc.
└── README.md
```

## Usage

To preview changes in your browser:

```bash
# mamba activate mkdocs
mkdocs serve
```

All pushes to main will render the content and publish to github pages automatically.
