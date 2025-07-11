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
global
  log stdout local0 info                         # Log level 'info' sent to stdout
  log stdout local1 notice                       # Log level 'notice' sent to stdout
  maxconn 250                                    # Maximum number of concurrent connections
  chroot /var/lib/haproxy                        # Restrict file system access for security
  stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
                                                 # Admin socket for monitoring and control
  stats timeout 60s                              # Timeout for the stats socket
  user haproxy                                   # Run HAProxy as user 'haproxy'
  group haproxy                                  # Run HAProxy as group 'haproxy'
  daemon                                         # Run as background service

defaults
  log global                                     # Inherit logging from 'global' section
  mode tcp                                       # Operate at TCP layer (Layer 4)
  retries 3                                      # Retry failed connections up to 3 times
  timeout connect 60s                            # Timeout for new connections
  timeout client 180s                            # Client inactivity timeout
  timeout server 180s                            # Server inactivity timeout
  option tcp-smart-connect                       # Smarter TCP connection handling

frontend pgsql
  mode tcp                                       # TCP mode for frontend
  bind *:5432                                    # Listen on all IPs, port 5432 (PostgreSQL)
  
  acl readonly.pgsql nbsrv(pgsql) eq 0           # If no servers available in 'pgsql', mark as read-only
  use_backend readonly.pgsql if readonly.pgsql   # Use read-only backend if condition is met

  acl pgsql nbsrv(pgsql) gt 0                    # If at least one server available in 'pgsql'
  use_backend pgsql if pgsql                     # Use primary backend if available

  default_backend pgsql                          # Fallback backend

backend pgsql
  mode tcp                                       # TCP mode
  balance roundrobin                             # Evenly distribute connections (even if only 1 server)
  option tcp-check                               # Enable health check via TCP
  server master-db 127.0.0.1:6432 check          # PgBouncer on localhost:6432

backend readonly.pgsql
  mode tcp                                       # TCP mode
  balance roundrobin                             # Even load balancing
  option tcp-check                               # Enable TCP-level health check
  server standby-db 127.0.0.1:6432 check          # Read-only server (PgBouncer)
```
