<div align="center">

[![Read Online: reth-explained](https://img.shields.io/badge/Read%20Online-reth--explained-blue?style=for-the-badge&logo=readthedocs)](https://reth-explained.netlify.app/)

</div>

## How to Run?

`mdbook` is a command-line tool based on Rust for creating books using `Markdown`. You need to install it locally first. For more details, refer to the official documentation: [mdbook](https://rust-lang.github.io/mdBook/)

First, install the required tools:

```bash
cargo install mdbook
```

To use Mermaid diagrams in your documentation, also install the mdbook-mermaid plugin:

```bash
cargo install mdbook-mermaid
```

If this is your first time adding mdbook-mermaid, initialize it in your book directory:

```bash
mdbook-mermaid install path/to/your/book
```

You can now use Mermaid syntax in your Markdown files to create diagrams, for example:

````markdown
```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```
````

Now you can serve the book locally:

```bash
mdbook serve
```

![mdbook](https://3bcaf57.webp.li/myblog/mdbook1.png)

## How to Participate?

We welcome contributions and community involvement! Here's how you can participate:

- **Report Issues:** If you find any mistakes, bugs, or have suggestions, please open an issue in this repository.
- **Submit Pull Requests:** Want to improve the content or fix something? Fork the repo, make your changes, and submit a pull request.
- **Join the Discussion:** Connect with us and other contributors in our [Telegram Community](https://t.me/+_a-io1KqMIc5ZjQ9).
- **Share Feedback:** Your feedback helps us improve! Feel free to comment on issues or PRs, or reach out via social channels.

Whether you're a beginner or an expert, your input is valuable. Let's build and learn together!

## How to Contribute

Ready to make a contribution? Follow these steps:

1. **Fork the Repository:** Click the "Fork" button at the top right of this page to create your own copy of the repository.
2. **Clone Your Fork:**
   ```bash
   git clone https://github.com/your-username/reth-explained.git
   cd reth-explained
   ```
3. **Create a New Branch:**
   ```bash
   git checkout -b your-feature-branch
   ```
4. **Make Your Changes:** Edit files, add content, or fix bugs as needed.
5. **Commit Your Changes:**
   ```bash
   git add .
   git commit -m "Clear and descriptive commit message"
   ```
6. **Push to Your Fork:**
   ```bash
   git push origin your-feature-branch
   ```
7. **Open a Pull Request:** Go to the original repository and click "New Pull Request". Select your branch and describe your changes.

**Best Practices:**
- Write clear, concise commit messages.
- Reference related issues in your PR description (e.g., "Closes #12").
- Make sure your code or content follows the project's style and guidelines.

Thank you for helping improve this project!

## About Youbet Study Group

[Youbet Study Group](https://x.com/youbetdao) is a hardcore web3 tech community focused on cryptography and blockchain infrastructure. For more tutorials, visit [Youbet Study Group Tutorials](https://according.work/tutorials).

If you want to learn more or join the group, please contact us!

- **Telegram Community**: [Join the discussion](https://t.me/+_a-io1KqMIc5ZjQ9)

**Special Thanks**
This project thanks [According.Work](https://according.work/) for technical support and collaboration.
