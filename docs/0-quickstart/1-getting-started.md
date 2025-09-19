# 🚀 Getting Started

Welcome! This section will walk you through how to get the project up and running on your local machine or development environment.

---

## 🧰 Prerequisites

Before you begin, ensure you have the following installed all the requirements. See the [Prerequisites](./0-prerequisites.md) section for detailed instructions on installing these tools.

---

## ✅ Walkthrough

After everything is wired up, you can run the following commands:

```bash
task dev # this one command will run all commands necessary to setup the environment
```

This will start the devbox environment and poetry environment and install all dependencies. And that is all you need to do to get started. (Yes, really.)

In a seperate terminal, run:

```bash
task docs # serve docs locally
```

Docs available at: [http://127.0.0.1:8000/](http://127.0.0.1:8000/)

All other commands are in the form of tasks. The project task file is `Taskfile.yaml`.

```bash
task --list-all # to see all project tasks
task <command> # usage
```

The project also uses gitflow for version control with gh-pages deployment automation. This is optional but you can also automate it using the `Taskfile.gitflow.yaml` file.

```bash
task -t Taskfile.gitflow.yaml --list-all # to see all gitflow tasks
task -t Taskfile.gitflow.yaml <command> # usage
```

See the [Tasks](../2-project/tasks/0-overview.md) section for more information on all tasks.

---

## 🧼 Cleanup

To tear everything down after testing:

```bash
task cleanup-dev # to cleanup everything running locally
task cleanup-prod # to cleanup everything running in production (IF YOU USED ANY PROD. WORKFLOWS)
task cleanup-all # to cleanup everything (local and production)
```

---

## ❓ Need Help?

If you get stuck:

* Check the [Troubleshooting](../3-troubleshooting/0-overview.md) guide.
* Open an issue on [GitHub](https://github.com/your-username/your-repo/issues)

---

Happy building! 🛠

---
