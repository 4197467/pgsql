# pgadmin

## Note de cours : PostgreSQL Admin

autres dépôts : http://framagit.org/jpython/meta


- wget http://monkey.org/~provos/libevent-1.4.14b-stable.tar.gz --no-check-certificate
- wget https://ftp.postgresql.org/pub/source/v9.6.1/postgresql-9.6.1.tar.bz2 --no-check-certificate
- wget https://github.com/tmux/tmux/archive/refs/tags/3.3a.tar.gz
- https://framagit.org/jpython/meta
- https://framagit.org/jpython/cours-postgresql-2/-/tree/master/atelier1
- https://use-the-index-luke.com/
---------------
- TODO :
- migrer slurm vers postgres sur les controleurs (master slave)
  - https://www.postgresql.org/docs/9.5/high-availability.html
- séparer moteur postgres des tablespaces (data)
- compiler postgres 9.6.1 pour edsr prod
-------------
```
select relname,last_vacuum, last_autovacuum, last_analyze, last_autoanalyze from pg_stat_user_tables;
TABLE pg_stat_activity;
```
