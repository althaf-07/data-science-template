# Data Science Template

```bash
sudo apt update
sudo apt install python3 git direnv
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```python
#!/usr/bin/python3

import json
import re
import readline  # noqa: F401
import shutil
import subprocess
import sys
from itertools import count  # noqa: F401
from pathlib import Path


def run_git_cmd(project_name, *commands):
    try:
        for command in commands:
            subprocess.run(["git", "-C", project_name] + command, check=True)
    except subprocess.CalledProcessError:
        print(f"Error: Failed to run {' '.join(command)}")
        sys.exit(1)


def is_valid_project_name(project_name):
    pattern = r"^[a-z\-]+$"
    return bool(re.match(pattern, project_name)) and 2 <= len(project_name) <= 50


def is_valid_author_name(author_name):
    pattern = r"^[A-Za-z ]+$"
    return bool(re.match(pattern, author_name)) and 2 <= len(author_name) <= 50


def clone_github_repo(CONFIG_DIR):
    command = [
        "git",
        "clone",
        "git@github.com:althaf-07/data-science-template.git",
        str(CONFIG_DIR / "data-science-template"),
    ]
    subprocess.run(
        command, check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL
    )
    shutil.rmtree(CONFIG_DIR / "data-science-template" / ".git")


def clone_or_reuse_template(CONFIG_DIR, project_name):
    template_path = CONFIG_DIR / "data-science-template"
    if template_path.is_dir():
        reuse_template = (
            input("Template exists. Reuse it? (Y/n): ").strip().lower() or "y"
        )
        if reuse_template != "y":
            shutil.rmtree(template_path)
            clone_github_repo(CONFIG_DIR)
    else:
        clone_github_repo(CONFIG_DIR)

    shutil.copytree(CONFIG_DIR / "data-science-template", project_name)


def get_project_name():
    project_name = (
        input("Enter the project's name (sample-project): ").strip() or "sample-project"
    )
    if not is_valid_project_name(project_name):
        print("Error: Invalid project name. Try again.")
        sys.exit(1)
    return Path(project_name)


def get_author_name(CONFIG_DIR):
    json_file = CONFIG_DIR / "mtds.json"
    if json_file.is_file():
        with json_file.open("r") as file:
            initial_author_name = json.load(file)["author_name"]
    else:
        initial_author_name = "John Doe"

    author_name = (
        input(f"Enter the author's name ({initial_author_name}): ").strip()
        or initial_author_name
    )
    if author_name != initial_author_name:
        if is_valid_author_name(author_name):
            json_file.parent.mkdir(parents=True, exist_ok=True)
            with json_file.open("w") as file:
                json.dump({"author_name": author_name}, file, indent=4)
        else:
            print("Error: Invalid author name. Try again.")
            sys.exit(1)
    return author_name


def initialize_git(project_name, remote_repo=None):
    run_git_cmd(
        project_name,
        ["init"],
        ["add", "."],
        ["commit", "-m", "Initial Commit"],
    )
    if remote_repo:
        run_git_cmd(project_name, ["remote", "add", "origin", remote_repo])
        run_git_cmd(project_name, ["push", "origin", "main"])


def get_dependencies(dependency_dict, message):
    selected = input(message).strip().split()
    dependencies = []
    for shorthand in selected[0]:
        if shorthand in dependency_dict:
            dependencies.append(dependency_dict[shorthand])
    return dependencies + selected[1:]


def initialize_uv(dev_env_dependencies, both_dependencies, project_name):
    commands = [
        ["uv", "add", "--dev"] + dev_env_dependencies,
        ["uv", "add"] + both_dependencies,
        ["uv", "sync", "--dev"],
    ]
    for command in commands:
        subprocess.run(command, cwd=project_name, check=True)


def main():
    CONFIG_DIR = Path.home() / ".config" / "mtds"

    project_name = get_project_name()
    clone_or_reuse_template(CONFIG_DIR, project_name)
    author_name = get_author_name(CONFIG_DIR)
    description = input("Enter project's description: ")
    remote_repo = input("Enter project's remote repository link: ")

    str_project_name = str(project_name)

    # File Editing
    with (project_name / "README.md").open("w") as file:
        file.write(
            "# "
            + " ".join(item.capitalize() for item in str_project_name.split("-"))
            + "\n"
        )
    (project_name / "gitignore").rename(project_name / ".gitignore")
    (project_name / "src" / "data_science_template").rename(
        project_name / "src" / str_project_name.replace("-", "_")
    )

    # pyproject.toml
    with (project_name / "pyproject.toml").open("r") as file:
        lines = file.readlines()
        lines[1] = lines[1].replace("data-science-template", str_project_name)
        if description:
            lines[3] = lines[3].replace("Add your description here", description)
        lines[4] = lines[4].replace("Author Name", author_name)
    with (project_name / "pyproject.toml").open("w") as file:
        file.writelines(lines)

    # Dependency handling
    dependency_dict = {
        "n": "numpy",
        "p": "pandas",
        "m": "matplotlib",
        "s": "seaborn",
        "l": "plotly",
        "y": "pyyaml",
        "d": "python-dotenv",
        "t": "pytest",
        "x": "xgboost",
        "g": "lightgbm",
        "k": "scikit-learn",
        "e": "ipykernel",
    }
    libs_with_shorthands = [
        dependency_dict[shorthand].replace(shorthand, f"[{shorthand}]", 1)
        for shorthand in dependency_dict
    ]
    dev_env_dependencies = get_dependencies(
        dependency_dict,
        f"List dependencies used only in Development Environment. {libs_with_shorthands}: ",
    )
    both_dependencies = get_dependencies(
        dependency_dict,
        f"List dependencies used in both Development and Production Environment. {libs_with_shorthands}: ",
    )

    if (project_name / ".envrc").is_file():
        subprocess.run(["direnv", "allow", str_project_name], check=True)

    # Install Development and Production dependencies using uv
    initialize_uv(dev_env_dependencies, both_dependencies, str_project_name)
    initialize_git(str_project_name, remote_repo)


if __name__ == "__main__":
    main()
```

Copy the above Python code and create a file named `mtds.py`.

```bash
sudo chmod +x mtds.py && sudo mv mtds.py /usr/local/bin/
```
