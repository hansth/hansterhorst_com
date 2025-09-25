# Hans ter Horst - Personal Website

This repository contains the source code for my personal website, [hansterhorst.com](https://www.hansterhorst.com), where I document and share my DevOps learning journey.

## About the Website

I'm a software developer with a background in Java and Angular, currently learning DevOps by creating my own HomeLab. This website serves as a platform to:

- Document my learning experiences.
- Share knowledge with the community.
- Organize my articles.
- Connect with others in the tech field.

## Technologies Used

- **[Hugo](https://gohugo.io/)**: A fast and modern static site generator.
- **Theme**: [Shibui](https://github.com/ntk148v/shibui).
- **Deployment**: [Cloudflare Pages](https://pages.cloudflare.com/).

## Local Development

### Prerequisites

- [Hugo CLI](https://gohugo.io/installation/).
- Git.

### Setup

1. Clone the repository:
   ```bash  
   git clone https://github.com/hansth/hansterhorst_com.git  
   cd hansterhorst_com
   ```
2. Initialize and update the theme submodule:
   ```bash  
   git submodule update --init --recursive  
   ```
3. Start the Hugo development server:
   ```bash  
   hugo server  
   ```
4. Open your browser and navigate to http://localhost:1313

## Deployment

This website is deployed using Cloudflare Pages with the following configuration:

- **Build command**: `hugo`
- **Build output directory**: `./public`
- **Custom domains**:
    - hansterhorst.com
    - www.hansterhorst.com

The deployment configuration is defined in the `wrangler.toml` file.

## Content Structure

- `/content/`: All website content.
    - `/_index.md`: Home page content.
    - `/about/`: About section.
    - `/posts/`: Blog posts.

## License

All rights reserved. This project and its contents are proprietary and confidential.

## Contact

- **Author**: Hans ter Horst
- **Website**: [hansterhorst.com](https://www.hansterhorst.com)