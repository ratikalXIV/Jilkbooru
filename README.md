# Hybooru

An edit to Hybooru specifically for a /xivg/-booru style board. To be hosted somewhere...


<p align="center">
   <img src="static/logo.svg" width="256"><br/>
   <a href="https://github.com/funmaker/Hybooru/releases/latest"><img src="https://github.com/funmaker/Hybooru/actions/workflows/release_build.yml/badge.svg"></a>
   <a href="https://github.com/funmaker/Hybooru/pkgs/container/hybooru"><img src="https://github.com/funmaker/Hybooru/actions/workflows/docker_build.yml/badge.svg"></a>
</p>

[Hydrus](https://github.com/hydrusnetwork/hydrus)-based booru-styled imageboard in React, inspired by [hyve](https://github.com/imtbl/hyve).

Demo: https://booru.funmaker.moe/

[API Documentation](API.md)

[Changelog](CHANGELOG.md)

Hybooru allows you to create an online booru-styled imageboard and REST API on top of Hydrus client,
allowing you to access your collection from anywhere, without the need for running Hydrus instance.
It uses its own PostgreSQL database, populated using metadata from Hydrus' SQLite database.
Files are not cloned and instead served directly from Hydrus's database. You need to
regenerate the Hybooru's database every time you want to update it. Make sure to
properly configure `configs.json` file. **Stop Hydrus when you regenerate HyBooru's
database if you plan to use live Hydrus' database (use hydrus backup instead if
possible)**


## Features

- Searching by tags
- Negative search
- Ratings
- Sorting (date imported, rating, size, etc)
- Searching tags and autocomplete
- tag and post relations (parents/siblings, duplicates/alternatives)
- Colored tags
- REST API
- Mobile support
- OpenGraph and OpenSearch
- Supports browsers without JS

Minimum Hydrus Version: **v447**

Keep in mind this project is not a standalone, fully-fledged booru, but rather a read-only interface to your Hydrus database.
It does not provide any way to manage your posts or tags. The only way to add/modify your data is to do these
changes in Hydrus and then rebuild Hybooru's database again(can be done from the cog menu on search/post page).

Currently, only Hydrus Client database is supported. You cannot use Hybooru on top of Hydrus Server.


## Setup

1) Install NodeJS, npm and PostgreSQL.
2) Create new Postgresql database and user.
3) Allow user to use `pg_trgm` and `intarray` extensions. Either:
   - Grant the user permission to create trusted extensions: `GRANT CREATE ON DATABASE <database> TO <user>`.
   - Create the extensions yourself: `CREATE EXTENSION IF NOT EXISTS pg_trgm; CREATE EXTENSION IF NOT EXISTS intarray;`.
4) Download latest [Release](https://github.com/funmaker/Hybooru/releases) production build [or build it yourself](#development).
5) Extract server files.
6) Edit `configs.json` to specify database credentials, hydrus db location and other options. See [Configuration](#configuration).
7) (Optional) Configure [reverse proxy](#reverse-proxy).
8) Run `npm install` to install dependencies.
9) Run `npm start` to start server.


## Searching Query Syntax

Searching tries to imitate classical booru's syntax. All tags are lowercase and use `_` instead of space character.
You can also use `?` to match for single character(eg: `?girl`) and `*` to match number of characters(eg: `blue_*`).
Patterns prefixed with `-` will be excluded from results. Patterns are matched against tag's name, but
Hydrus's `namespace:subtag` syntax is also supported.

Additionally you can sort results by including `order:*` in query. Supported sorts are: `order:posted`(date),
`order:id`, `order:rating`, `order:size`. You can also append `_desc` or `_asc` to specify order(eg: `order:posted_asc`).
If not specified, post are sorted by date descending.

If you use a numeric rating service and successfully imported the ratings, you can also filter posts by their ratings
using `rating:` namespace. You can search posts with specific rating(`rating:3`), range(`rating:2-4`) or query posts
that have not been rated(`rating:none`).

Eg: `1girl blue_* -outdoors rating:3-5 order:rating_desc`


## Configuration

Hybooru's config is stored in `configs.json` file in the project's root directory. Restart Hybooru to apply changes.

| Name                         | Type                      | Default                                           | Comment                                                                                                                                                                                        |
|------------------------------|---------------------------|---------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| port                         | number                    | `3939`                                            | HTTP server port.                                                                                                                                                                              |
| hydrusDbPath                 | string or null            | `null`                                            | Hydrus db or backup location. If null, default platform-dependent locaton is used: `%appdata%/hydrus/db`(Windows), `~/.local/share/hydrus/db`(Linux), `~/Library/Preferences/hydrus/db`(MacOS) |
| appName                      | string                    | `"Hybooru"`                                       | Specify name of your booru (appears as logo).                                                                                                                                                  |
| appDescription               | string                    | `"Hydrus-based booru-styled imageboard in React"` | Booru's description used in OpenGraph.                                                                                                                                                         |
| adminPassword                | string or null            | `null`                                            | Password used to regenerate database (can be accessed from the cog button). Null disables manual database regeneration.                                                                        |
| isTTY                        | boolean or null           | `null`                                            | Overrides colorful/fancy output. `true` forces, `false` disables, `null` automatically determines. Useful when piping output.                                                                  |
| importBatchSize              | number                    | `8192`                                            | Base batch size used during importing. Decrease it if hybooru crashes during import.                                                                                                           |
| db                           | PoolConfig                | _local database_                                  | node-postgres config object. See https://node-postgres.com/api/client for more details. By defaults it attempts to connect to `hybooru` database at `localhost` using `hybooru` as password.   |
| posts                        | object                    | _see below_                                       | Options related to posts and files.                                                                                                                                                            |
| posts.services               | (string/number)[] or null | `null`                                            | List of names or ids of file services to import. Use `null` to import from all services.                                                                                                       |
| posts.filesPathOverride      | string or null            | `null`                                            | Overrides location of post's files. If `null`, `client_files` inside hydrus's db folder is used.                                                                                               |
| posts.thumbnailsPathOverride | string or null            | `null`                                            | Overrides location of post's thumbnails. If `null`, `filesPathOverride` is used.                                                                                                               |
| posts.pageSize               | number                    | `72`                                              | Number of posts on single page.                                                                                                                                                                |
| posts.cachePages             | number                    | `5`                                               | Number of pages cached in single cache entry.                                                                                                                                                  |
| posts.cacheRecords           | number                    | `1024`                                            | Max number of cache entries.                                                                                                                                                                   |
| posts.maxPreviewSize         | number                    | `104857600`                                       | Max size in bytes of post that can be previewed in post page/gallery. Default is 100MB.                                                                                                        |
| tags                         | object                    | _see below_                                       | Options related to tags. All tags below support wildcards.                                                                                                                                     |
| tags.services                | (string/number)[] or null | `null`                                            | List of names or ids of tag services to import. Use `null` to import from all services.                                                                                                        |
| tags.motd                    | string or object or null  | `null`                                            | Tag used to for random image displayed on main page. You can also specify object to specify different tags for different themes(use `light`, `dark` and `auto` as keys)                        |
| tags.untagged                | string                    | `"-*"`                                            | Overrides query used to determine which posts require tagging. Default `"-*"` matches all posts with no tags.                                                                                  |
| tags.ignore                  | string[]                  | `[]`                                              | List of tags that will not be imported from Hydrus (posts tagged by these tags will still be imported).                                                                                        |
| tags.blacklist               | string[]                  | `[]`                                              | All posts and tags matching any of specified tags will not be imported from Hydrus.                                                                                                            |
| tags.whitelist               | string[] or null          | `null`                                            | Only posts matching specified tags will be imported from Hydrus. Use `null` or empty array to ignore whitelist.                                                                                |
| tags.resolveRelations        | boolean                   | `true`                                            | Resolve tag siblings and parents. Can be slow in large databases.                                                                                                                              |
| tags.reportLoops             | boolean                   | `false`                                           | Print out all loops detected in tag relationships.                                                                                                                                             |
| tags.searchSummary           | number                    | `39`                                              | Number of tags that appear on side menu when searching posts.                                                                                                                                  |
| rating                       | object or null            | _see below_                                       | Options related to numerical rating. Set null to remove ratings.                                                                                                                               |
| rating.enabled               | boolean                   | `true`                                            | Enables or disables rating import.                                                                                                                                                             |
| rating.stars                 | number                    | `5`                                               | Number of stars used in rating.                                                                                                                                                                |
| rating.serviceName           | string or null            | `null`                                            | Name of the numerical rating service name. Set to null to pick any service.                                                                                                                    |
| versionCheck                 | object or null            | _see below_                                       | Options related to version checking. Set null to disable.                                                                                                                                      |
| versionCheck.enabled         | boolean                   | `true`                                            | Enables or disables version checking.                                                                                                                                                          |
| versionCheck.owner           | string                    | `"funmaker"`                                      | GitHub handle of the repo owner. Do not change unless you know what you are doing.                                                                                                             |
| versionCheck.repo            | string                    | `"hybooru"`                                       | GitHub handle of the repo name. Do not change unless you know what you are doing.                                                                                                              |
| versionCheck.cacheLifeMs     | number                    | `3600000` (1 hour)                                | Lifetime of versions cache. GitHub API is rate-limited, do not change unless you know what you are doing.                                                                                      |

## Development

Build scripts are written for Linux. Building on Windows is currently not supported.
However, you can still look into `package.json` and change `scripts` to use Window's commands.
Alternatively you can probably just use Docker or a virtual machine.

### Install Dependencies

```bash
npm install
```

### Run development

```bash
npm run start
```

### Build production

```bash
npm run build:prod
```

Output is saved to `dist/` folder in project's root directory. These are the files you will want to deploy.

### Start production

```bash
cd dist
npm start
```


## Reverse Proxy

It is recommended to set up a reverse proxy to serve static files and enable HTTPS.

Example Nginx configuration:

```nginx
server {
    listen 80;
    listen 443 ssl;
    
    server_name booru.example.com;
    
    # Uncomment to override thumbnails location
    #location ~ ^\/files\/t(..)(.*)$ { 
    #    root /path/to/thumbnails;
    #    try_files /t$1/$1$2 =404;
    #}
    
    location ~ ^\/files\/(.)(..)(.*)$ {
        root /path/to/files; # hydrus's files location (eg: client_files folder inside hydrus's db folder)
        try_files /$1$2/$2$3 =404;
    }
    
    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        proxy_pass http://localhost:3939/;
    }
}
```
