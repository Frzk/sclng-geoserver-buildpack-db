# GeoServer buildpack

This buildpack aims at installing a GeoServer instance. The instance is
deployed with a database addon and two plugins to persist data across
deployments (`JDBCConfig` and `JDBCStore`).

## Usage

The following instructions will help you deploy your own GeoServer instance:

1. In the working directory of your choice, initialize a `git` repository:

```bash
% mkdir scalingo-geoserver
% cd scalingo-geoserver
% git init
```

2. Create a file named `.buildpacks` with the following content:

```
https://github.com/Scalingo/geoserver-buildpack.git
https://github.com/Scalingo/java-war-buildpack.git
```

This instructs Scalingo to use 2 buildpacks:

- the first one is used to download and install GeoServer and its plugins.
- the second one is used to run your GeoServer instance.

3. Create a file named `start.sh` with the following content:

```bash
#!/usr/bin/env bash

# Parse SCALINGO_POSTGRESQL_URL env var:
host=$( echo $SCALINGO_POSTGRESQL_URL | cut -d "@" -f2 | cut -d ":" -f1 )
username=$( echo $SCALINGO_POSTGRESQL_URL | cut -d "/" -f3 | cut -d ":" -f1 )
port=$( echo $SCALINGO_POSTGRESQL_URL | cut -d ":" -f4 | cut -d "/" -f1 )
password=$( echo $SCALINGO_POSTGRESQL_URL | cut -d "@" -f1 | cut -d ":" -f3 )
database=$( echo $SCALINGO_POSTGRESQL_URL | cut -d "?" -f1 | cut -d "/" -f4 )

make_jdbc_plugin_config() {
    local dir
    local config_dir
    local config_file
    local init_database

    dir="${1}"

    printf -v config_dir "%s/%s" "${GEOSERVER_DATA_DIR}" "${dir}"
    printf -v config_file "%s/%s.properties" "${config_dir}" "${dir}"

    init_database="false"

    if [[ -n "${GEOSERVER_INIT}" ]]
    then
        init_database="true"
    fi

    mkdir -p "${config_dir}"

    cat << EOF > "${config_file}"
enabled=true
initdb=${init_database}
initScript=${config_dir}/init.sql
import=false
jdbcUrl=jdbc:postgresql://${host}:${port}/${database}
username=${username}
password=${password}
driverClassName=org.postgresql.Driver
EOF
}

# Create config files for the JDBC* plugins:
make_jdbc_plugin_config "jdbcconfig"
make_jdbc_plugin_config "jdbcstore"

# Start Geoserver:
#java_opts="-Xmx384m -Xss512k -XX:+UseCompressedOops"
java -jar /app/webapp-runner.jar --port ${PORT} /app/geoserver.war
```

This file will be used by Scalingo to start your GeoServer instance.
It will also initialize the database used by GeoServer if needed.

4. Create a file named `Procfile` with the followin content:

```
web: bash start.sh
```

5. Create a the following directory and subdirectories to store GeoServer configuration files:

```
% mkdir -p geoserver-data/{jdbcconfig,jdbcstore}
```

6. Download database initialization scripts:

```bash
% curl \
    --location \
    https://raw.githubusercontent.com/geoserver/geoserver/main/src/community/jdbcconfig/src/main/resources/org/geoserver/jdbcconfig/internal/initdb.postgres.sql \
    --output geoserver-data/jdbcconfig/init.sql

% curl \
    --location \
    https://raw.githubusercontent.com/geoserver/geoserver/main/src/community/jdbcstore/src/main/resources/org/geoserver/jdbcstore/internal/init.postgres.sql \
    --output geoserver-data/jdbcstore/init.sql
```

7. Validate your changes:

```bash
% git add .
% git commit -m "First commit"
```

8. Create your application:

```bash
% scalingo create my-geoserver
% scalingo --app my-geoserver scale web:1:M
% scalingo --app my-geoserver addons-add postgresql postgresql-business-512
```

9. Prepare GeoServer's environment:

```bash
% scalingo --app my-geoserver env-set GEOSERVER_DATA_DIR="/app/geoserver-data"
% scalingo --app my-geoserver env-set GEOSERVER_INIT="yes"
```

10. Deploy:

```bash
% git push scalingo main
```

11. Head to your brand new GeoServer, configure it, make sure to change the
admin account password, etc.

12. Now that the database has been initialized, remove the `GEOSERVER_INIT`
environment variable:

```bash
% scalingo --app my-geoserver env-unset GEOSERVER_INIT
% scalingo --app my-geoserver restart
```

This will prevent GeoServer to try to (re-)initialize the database when
deploying again in the future (which would fail).


