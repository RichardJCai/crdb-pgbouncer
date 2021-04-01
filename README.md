# Setting up CockroachDB with pgbouncer + HAProxy
### Insecure

1. First setup CockroachDB Cluster
    - https://www.cockroachlabs.com/docs/stable/start-a-local-cluster.html
2.  Setup HAProxy for CRDB Cluster
    - https://www.cockroachlabs.com/docs/v20.2/deploy-cockroachdb-on-premises-insecure#step-5-set-up-load-balancing
    Sample haproxy.cfg
    ```
    global
    maxconn 4096

    defaults
        mode                tcp

        # Timeout values should be configured for your specific use.
        # See: https://cbonte.github.io/haproxy-dconv/1.8/configuration.html#4-timeout%20connect

        # With the timeout connect 5 secs,
        # if the backend server is not responding, haproxy will make a total
        # of 3 connection attempts waiting 5s each time before giving up on the server,
        # for a total of 15 seconds.
        retries             2
        timeout connect     5s

        # timeout client and server govern the maximum amount of time of TCP inactivity.
        # The server node may idle on a TCP connection either because it takes time to
        # execute a query before the first result set record is emitted, or in case of
        # some trouble on the server. So these timeout settings should be larger than the
        # time to execute the longest (most complex, under substantial concurrent workload)
        # query, yet not too large so truly failed connections are lingering too long
        # (resources associated with failed connections should be freed reasonably promptly).
        timeout client      10m
        timeout server      10m

        # TCP keep-alive on client side. Server already enables them.
        option              clitcpka

    listen psql
        bind :26257
        mode tcp
        balance roundrobin
        option httpchk GET /health?ready=1
        server cockroach1 $ip:26257 check port 26258
        server cockroach2 $ip:26257 check port 26258
        server cockroach3 $ip:26257 check port 26258
    ```
3. Setup pgbouncer (pgbouncer can be run on a CRDB node or a separate node)
    - Install pgbouncer 
        - MacOS: `brew install pgbouncer`
        - Ubuntu: `apt-get install pgbouncer`
    - Create pgbouncer.ini config
    - Sample ini
        ```
        [databases]
        db_name = host=$ip_to_haproxy port=ha_proxy_port db_name=tpcc

        [pgbouncer]
        pool_mode = session
        listen_port = 6432
        listen_addr = *
        auth_type = trust
        auth_file = userlist.txt
        logfile = pgbouncer.log
        pidfile = pgbouncer.pid
        admin_users = root
        ignore_startup_parameters = extra_float_digits
        client_tls_sslmode = disable
        server_reset_query = false
        max_client_conn = 200
        default_pool_size = 25
        ```
    - Important to note, dbname has to be specified in the config, however in CRDB, you can switch dbs even after connecting to a different db in the same session
    - auth_type trust is used for insecure meaning no authentication is done
    - Accounts you wish to use must be specified in userlist.txt
        - Even for auth_type trust, your user must be listed
        - Example userlist.txt
            ```
            "root" ""
            "testuser" ""
            ```
    - ignore_startup_parameters = extra_float_digits must be specified or connections will error
4. Run `pgbouncer pgbouncer.ini`
5. Connect to db using pgbouncer
    - psql -U root -p 6432 -h $ip_of_pgbouncer_machine db_name
    - cockroach sql --url 'postgres://$user@$ip_of_pgbouncer_machine:6433/db_name?sslmode=disable'

### Secure
1. First setup CockroachDB Cluster
    - https://www.cockroachlabs.com/docs/v20.2/secure-a-cluster
2.  Setup HAProxy for CRDB Cluster
    - https://www.cockroachlabs.com/docs/v20.2/deploy-cockroachdb-on-premises#step-6-set-up-load-balancing
    Sample haproxy.cfg
3. Setup pgbouncer (pgbouncer can be run on a CRDB node or a separate node)
    - Install pgbouncer 
        - MacOS: `brew install pgbouncer`
        - Ubuntu: `apt-get install pgbouncer`
    - Create pgbouncer.ini config
    ```
    [databases]
    db_name = host=localhost port=26257 dbname=db_name

    [pgbouncer]
    pool_mode = session
    listen_port = 6433
    listen_addr = *
    auth_type = cert
    auth_file = userlist.txt
    logfile = pgbouncer.log
    admin_users = root
    ignore_startup_parameters = extra_float_digits
    server_reset_query = false
    max_client_conn = 200
    client_tls_sslmode = verify-full
    client_tls_key_file = ./pgbouncer-certs/ca.key
    client_tls_cert_file = ./pgbouncer-certs/ca.crt
    client_tls_ca_file = ./pgbouncer-certs/ca.crt

    server_tls_sslmode = verify-full
    server_tls_ca_file = ./certs/ca.crt
    ```
    - Notes: auth_type cert is specified
        - `Client must connect over TLS connection with a valid client certificate. The user name is then taken from the CommonName field from the certificate.`
    - `client_tls_sslmode = verify=full` specifies that connections to PGBouncer use TLS, certs must be created for connections to PGBouncer
        - `cockroach cert create-ca \
 --certs-dir=[path-to-certs-directory] \
 --ca-key=[path-to-ca-key]` can be used to create the CA cert and key
    - `server_tls_sslmode = verify-full` specifies that connections from PGBouncer to CRDB must connect over TLS. 
        - Follow https://www.cockroachlabs.com/docs/v20.2/deploy-cockroachdb-on-premises#step-2-generate-certificates to get the relevant certs to specify
    - Accounts you wish to use must be specified in userlist.txt
        - Example userlist.txt
            ```
            "root" ""
            "testuser" ""
            ```
4. Run `pgbouncer pgbouncer.ini`
5. Connect to db using pgbouncer
    - cockroach sql --url 'postgres://$user@$ip_of_pgbouncer_machine:6433/db_name?sslmode=require' --certs-dir
        - The required certs must be made using the PGBouncer CA.
        - On the PGBouncer machine, run 
            ```
            cockroach cert create-client    [username] --certs-dir [path-to-certs-directory] --ca-key=[path-to-ca-key]
            ```
            -  In step 3, you created certs for PGBouncer
        - client certs will be created for the specified user, transfer these certs to where you're making the connection from

### Suggestions: 
- CockroachDB is compatible with PGBouncer.
- In CRDB, idle connections are fairly cheap ~20000-30000 byte overhead per connection. PGBouncer is likely unnecessary but may be used for application side benefits. 
- PGBouncer may be useful if hundreds of thousands of connections are being created and may improve performance depending on the hardware specifications relative to the number of connections. 


#### Known issues:
- Prepared statements will not reset on connections, this is a general PGBouncer issue.
- From the PGBouncer FAQ: https://www.pgbouncer.org/faq.html
    ```
    How to use prepared statements with session pooling?
    In session pooling mode, the reset query must clean old prepared statements. This can be achieved by server_reset_query = DISCARD ALL; or at least to DEALLOCATE ALL;

    How to use prepared statements with transaction pooling?
    To make prepared statements work in this mode would need PgBouncer to keep track of them internally, which it does not do. So the only way to keep using PgBouncer in this mode is to disable prepared statements in the client.
    ```
