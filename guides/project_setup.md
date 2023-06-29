# Setup project on your machine

If you see files like `docker-compose.yml`, `Dockerfile` or `dip.yml` in your project root, please use Docker for local development.

## Local development with Docker

1. Make sure Docker is installed on your machine. If not please visit [official website](https://docs.docker.com/get-docker/) and download it.
2. Run

```bash
docker compose up --build rails
```

or `docker-compose` for older Docker versions, to build application and its dependencies.

The main service with web application can have different name, so please check `docker-compose.yml` at first. In some projects we are using **app** as service name.

3. If you've started **rails** service in background, using `-d` flag, you can check logs with this command.

```bash
docker compose logs -f rails
```

4. If you are using RubyMine, you can refer to this [guide](https://www.jetbrains.com/help/ruby/using-docker-compose-as-a-remote-interpreter.html), for VSCode - please check this [one](https://code.visualstudio.com/docs/remote/containers).
5. To prepare database for local development, run

```bash
docker-compose exec rails bundle exec rails db:setup
```

6. If you want to restore database from backup, run

```bash
cat your_dump.sql | docker exec -i {docker-postgres-container} psql -U {user} -d {database_name}
```

Your project may use [dip](https://github.com/bibendi/dip) to simplify Docker commands.

For example instead of `docker compose up rails` you will use `dip rails s` and result will be the same. You need to install **dip** gem at first with `gem install dip`.
You can check all available commands in **dip.yml** file in project root or using `dip ls` command. Please refer to this [guide](https://evilmartians.com/chronicles/ruby-on-whales-docker-for-ruby-rails-development) for more information.

## Local development with RVM

1. Make sure you have [rvm](https://rvm.io/) installed. We use it for keeping different ruby versions for each project. You can find ruby version for your project in **.ruby-version** and gemset to isolate gems used in project in **.ruby-gemset**.

2. Make sure you have [PostgreSQL](https://www.postgresql.org/download/) and [Redis](https://redis.io/topics/quickstart) on your system.

3. Make sure you have Node.js installed. If you see .nvmrc file in your project directory, install [nvm](https://github.com/nvm-sh/nvm#install--update-script) and corresponding node version.

4. Install [lefthook](https://github.com/evilmartians/lefthook). We use it for pre-commit hooks and commit message validation. Don't forget to copy **lefthook.yml** configuration to your project directory if it is missing (Will be attached below).

5. Clone your project from Gitlab (even if it is empty directory).

6. Run

```bash
rvm install
```

from your console inside project directory.

7. Pick your ruby version and gemset in `Languages & Frameworks → Ruby SDK` (Only if you are using RubyMine)

8. Install correct version of **bundler**. You can find it at the end of **Gemfile.lock**. Command to install: `gem install bundler:$BUNDLE_VERSION`

9. Run

```bash
bundle check || bundle install
```

10. Look at **database.example.yml** and **secrets.example.yml** and create new ones according to examples (without .example extension).
    If your project using **credentials.yml** please ask Team Lead for **master.key**.

11. Setup database with

```bash
bundle exec rails db:setup
```

12. Launch server with

```bash
bundle exec rails s
```

or if you have **Procfile** inside your project, you should run

```bash
foreman start
```

It will launch not only rails server, but other processes needed for development. For example: webpacker to build Vue.js front-end.

## References
- https://evilmartians.com/chronicles/ruby-on-whales-docker-for-ruby-rails-development
- https://github.com/bibendi/dip
- https://github.com/rails/docked
- https://rvm.io/