# NAME

PostgreSQL::Test::Session - class for a PostgreSQL libpq session

# SYNOPSIS

     use PostgreSQL::Test::Session;
    
     use PostgreSQL::Test::Cluster;

     my $node = PostgreSQL::Test::Cluster->new('mynode');

     # create a new session. defult dbname is 'postgres'
     my $session = PostgreSQL::Test::Session->new(node => $node 
                                                  [, dbname => $dbname] );

     # close the session
     $session->close;

     # reopen the session, after closing it if not closed
     $session->reconnect;

     # check if the session is ok
     # my $status = $session->conn_status;

     # run some SQL, not producing tuples
     my $result = $session->do($sql, ...);

     # run an SQL statement asynchronously
     my $result = $session->do_async($sql);

     # wait for and async SQL to complete
     $session->wait_for_completion;

     # set a password for a user
     my $result = $session->set_password($user, $password);

     # get some data
     my $result = $session->query($sql);

     # get a single value, default croaks if no value found
     my $val = $session->query_oneval($sql [, $missing_ok ]);

     #return lines of tuples like "psql -A -t"
     my @lines = $session->query_tuples($sql, ...);

# DESCRIPTION

PostgreSQL::Test::Session encapsulates a libpq session for use in PostgreSQL
TAP tests, allowing the test to connect without having to spawn \`psql\` in a
child process. 

The session object is automatically closed when the object goes out of scope,
including at script end.

Several methods return a hashref as a result, which will have the following
fields:

> The last 4 will be empty unless the SQL produces tuples.

# METHODS

- `PostgreSQL::Test::Session-`new(node=> $node \[, dbname=> $dbname \])>

    Set up a new session for the node, whhich must be a PostgreSQL::Test::Cluster
    instance. The default dbame is 'postgres'

- `` PostgreSQL::Test::Session->new(connstr => $connstr \[, libdir => $libdir\]) >>

    Set up a new session for the connection string. If using the FFI libpq wrapper,
    $libdir must point to the directory where the libpq library is installed.

- `$session->close()`

    Close the connection

- `$session->reconnect()`

    Reopen the session using the original connstr. If the session is still open,
    close it before reopening.

- `$session->conn_status()`

    Return the connection status. This will be a libpq status value like
    CONNECTION\_OK.

- `$session->do($sql, ...)`

    Run one or more SQL statements synchronously (using PQexec). The statements
    should not return any tuples. Returns the status, which will be
    PGRES\_COMMAND\_OK (i.e. 1) in the case of success.

- `$session->do_async($sql)`

    Run a single statement asynchronously, using PQsendQuery. The return value
    is a boolean indicating success.

- `$session->wait_for_completion()`

    Wait until all asynchronous SQL has completed

- `$session->set_password($user, $password)`

    Set the user's password by calling PQchangePassword.

    Returns a result hash.

- `$session->query($sql)`

    Runs sql that might return tuples.

    Returns a result hash.

- `$session->query_oneval($sql [, $missing_ok ] )`

    Run a query that is expected to return no more than one tuple with one value;

    If $missing\_ok is true, return undef if the query returns no tuple. Otherwise
    croak if there is not exactly one tuple, or of the tuple does not have
    exctly one value.

    If none of these apply, return the single value from the query. A NULL value
    will result in undef, so if $missing\_ok is true you won't be able to
    distinguish between a null value and a missing tuple.

    A non NULL value is returned as the string value obtained from PQgetvalue.

- `$session->query_tuples($sql, ...)`

    Run the sql commands and return the output as a single piece of text in the
    same format as psql -A -t.

    Fields within tuples are separated by a "|", tuples are spearated by "\\n"

# POD ERRORS

Hey! **The above document had some coding errors, which are explained below:**

- Around line 64:

    &#x3d;over should be: '=over' or '=over positive\_number'

- Around line 129:

    You forgot a '=back' before '=head1'
