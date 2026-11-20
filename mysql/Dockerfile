# MariaDB instead of mysql:5.7:
#  - mysql:5.7 is EOL and amd64-only (fails on ARM Ubuntu)
#  - on Ubuntu hosts that have MySQL installed, the host AppArmor profile for
#    /usr/sbin/mysqld also confines the container's mysqld, so `podman stop`
#    cannot signal it and the container hangs in "Stopping". MariaDB's binary
#    is /usr/sbin/mariadbd, which that profile does not match.
FROM docker.io/library/mariadb:10.11

ENV MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=yes \
    MARIADB_DATABASE=cities \
    MARIADB_USER=shipping \
    MARIADB_PASSWORD=secret

COPY scripts/* /docker-entrypoint-initdb.d/
