# 📃 Git Clone without Ancestry

My website, [https://codingindex.xyz](https://codingindex.xyz) is run on GitHub pages. Whenever I write a blog post and push to the repository, a GitHub actions pulls that new blog post, does some processing, and publishes the resulting static site in the same repository.

Needless to say, this means that the repository is _huge._ Hence, on a new computer, it isn't really wise to clone the entire repository, because there is just no need to maintain ancestry.

Luckily, `git` has a way to clone without getting every single commit since the project started: `git clone <git url> --depth=1 --single-branch`. When doing `git log`, you will see that the latest commit has become `grafted`, which lets your repository pretend that it doesn't have parents past the latest commit (source: [https://stackoverflow.com/questions/27296188/what-exactly-is-a-grafted-commit-in-a-shallow-clone](https://stackoverflow.com/questions/27296188/what-exactly-is-a-grafted-commit-in-a-shallow-clone)).
