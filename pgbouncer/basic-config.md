High Availability Architecture (Single-Master with Failover)
```
Client  
│  
▼  
HAProxy (TCP or HTTP load balancing)  
│  
▼  
PgBouncer (in session / transaction mode)  
│  
▼  
PostgreSQL:  
├── Primary node (active)  
└── Standby node (repmgr - streaming replication)
```
Concepts
```
HAProxy: High-performance load balancer for TCP and HTTP protocols
PgBouncer: Connection pooler
repmgr: Replication management and monitoring tool for PostgreSQL
```
Config /etc/haproxy/haproxy.cfg
```
/etc/haproxy/haproxy.cfg
global
log stdout local0 info
log stdout local1 notice
maxconn 250
chroot /var/lib/haproxy
stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
stats timeout 60s
user haproxy
group haproxy
daemon

defaults
log global
mode tcp
retries 3
timeout connect 60s
timeout client 180s
timeout server 180s
option tcp-smart-connect

frontend pgsql
mode tcp
bind \*:5432
acl readonly.pgsql nbsrv(pgsql) eq 0
use_backend readonly.pgsql if readonly.pgsql
acl pgsql nbsrv(pgsql) gt 0
use_backend pgsql if pgsql
default_backend pgsql

backend pgsql
mode tcp
balance roundrobin
option tcp-check
server master-db 127.0.0.1:6432 check

backend readonly.pgsql
mode tcp
balance roundrobin
option tcp-check
server standby-db 127.0.0.1:6432 check
```
