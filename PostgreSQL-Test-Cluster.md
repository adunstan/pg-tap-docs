# NAME

PostgreSQL::Test::Cluster - class representing PostgreSQL server instance

# SYNOPSIS

    use PostgreSQL::Test::Cluster;

    my $node = PostgreSQL::Test::Cluster->new('mynode');

    # Create a data directory with initdb
    $node->init();

    # Start the PostgreSQL server
    $node->start();

    # Add a setting and restart
    $node->append_conf('postgresql.conf', 'hot_standby = on');
    $node->restart();

    # Modify or delete an existing setting
    $node->adjust_conf('postgresql.conf', 'max_wal_senders', '10');

    # get pg_config settings
    # all the settings in one string
    $pgconfig = $node->config_data;
    # all the settings as a map
    %config_map = ($node->config_data);
    # specified settings
    ($incdir, $sharedir) = $node->config_data(qw(--includedir --sharedir));

    # run a query with psql, like:
    #   echo 'SELECT 1' | psql -qAXt postgres -v ON_ERROR_STOP=1
    $psql_stdout = $node->safe_psql('postgres', 'SELECT 1');

    # Run psql with a timeout, capturing stdout and stderr
    # as well as the psql exit code. Pass some extra psql
    # options. If there's an error from psql raise an exception.
    my ($stdout, $stderr, $timed_out);
    my $cmdret = $node->psql('postgres', 'SELECT pg_sleep(600)',
            stdout => \$stdout, stderr => \$stderr,
            timeout => $PostgreSQL::Test::Utils::timeout_default,
            timed_out => \$timed_out,
            extra_params => ['--single-transaction'],
            on_error_die => 1)
    print "Sleep timed out" if $timed_out;

    # Similar thing, more convenient in common cases
    my ($cmdret, $stdout, $stderr) =
        $node->psql('postgres', 'SELECT 1');

    # run query every second until it returns 't'
    # or times out
    $node->poll_query_until('postgres', q|SELECT random() < 0.1;|')
      or die "timed out";

    # Do an online pg_basebackup
    my $ret = $node->backup('testbackup1');

    # Take a backup of a stopped server
    $node->stop;
    my $ret = $node->backup_fs_cold('testbackup3')

    # Restore it to create a new independent node (not a replica)
    my $other_node = PostgreSQL::Test::Cluster->new('mycopy');
    $other_node->init_from_backup($node, 'testbackup');
    $other_node->start;

    # Stop the server
    $node->stop('fast');

    # Find a free, unprivileged TCP port to bind some other service to
    my $port = PostgreSQL::Test::Cluster::get_free_port();

# DESCRIPTION

PostgreSQL::Test::Cluster contains a set of routines able to work on a PostgreSQL node,
allowing to start, stop, backup and initialize it with various options.
The set of nodes managed by a given test is also managed by this module.

In addition to node management, PostgreSQL::Test::Cluster instances have some wrappers
around Test::More functions to run commands with an environment set up to
point to the instance.

The IPC::Run module is required.

# METHODS

- $node->port()

    Get the port number assigned to the host. This won't necessarily be a TCP port
    open on the local host since we prefer to use unix sockets if possible.

    Use $node->connstr() if you want a connection string.

- $node->host()

    Return the host (like PGHOST) for this instance. May be a UNIX socket path.

    Use $node->connstr() if you want a connection string.

- $node->basedir()

    The directory all the node's files will be within - datadir, archive directory,
    backups, etc.

- $node->name()

    The name assigned to the node at creation time.

- $node->logfile()

    Path to the PostgreSQL log file for this instance.

- $node->connstr()

    Get a libpq connection string that will establish a connection to
    this node. Suitable for passing to psql, DBD::Pg, etc.

- $node->group\_access()

    Does the data dir allow group access?

- $node->data\_dir()

    Returns the path to the data directory. postgresql.conf and pg\_hba.conf are
    always here.

- $node->archive\_dir()

    If archiving is enabled, WAL files go here.

- $node->backup\_dir()

    The output path for backups taken with $node->backup()

- $node->install\_path()

    The configured install path (if any) for the node.

- $node->pg\_version()

    The version number for the node, from PostgreSQL::Version.

- $node->config\_data( option ...)

    Return configuration data from pg\_config, using options (if supplied).
    The options will be things like '--sharedir'.

    If no options are supplied, return a string in scalar context or a map in
    array context.

    If options are supplied, return the list of values.

- $node->info()

    Return a string containing human-readable diagnostic information (paths, etc)
    about this node.

- $node->dump\_info()

    Print $node->info()

- $node->init(...)

    Initialize a new cluster for testing.

    Authentication is set up so that only the current OS user can access the
    cluster. On Unix, we use Unix domain socket connections, with the socket in
    a directory that's only accessible to the current user to ensure that.
    On Windows, we use SSPI authentication to ensure the same (by pg\_regress
    \--config-auth).

    WAL archiving can be enabled on this node by passing the keyword parameter
    has\_archiving => 1. This is disabled by default.

    postgresql.conf can be set up for replication by passing the keyword
    parameter allows\_streaming => 'logical' or 'physical' (passing 1 will also
    suffice for physical replication) depending on type of replication that
    should be enabled. This is disabled by default.

    force\_initdb => 1 will force the initialization of the cluster with a new
    initdb rather than copying the data folder from a template.

    The new node is set up in a fast but unsafe configuration where fsync is
    disabled.

- $node->append\_conf(filename, str)

    A shortcut method to append to files like pg\_hba.conf and postgresql.conf.

    Does no validation or sanity checking. Does not reload the configuration
    after writing.

    A newline is automatically appended to the string.

- $node->adjust\_conf(filename, setting, value, skip\_equals)

    Modify the named config file setting with the value. If the value is undefined,
    instead delete the setting. If the setting is not present no action is taken.

    This will write "$setting = $value\\n" in place of the existing line,
    unless skip\_equals is true, in which case it will write
    "$setting $value\\n". If the value needs to be quoted it is the caller's
    responsibility to do that.

- $node->backup(backup\_name)

    Create a hot backup with **pg\_basebackup** in subdirectory **backup\_name** of
    **$node->backup\_dir**, including the WAL.

    By default, WAL files are fetched at the end of the backup, not streamed.
    You can adjust that and other things by passing an array of additional
    **pg\_basebackup** command line options in the keyword parameter backup\_options.

    You'll have to configure a suitable **max\_wal\_senders** on the
    target server since it isn't done by default.

- $node->backup\_fs\_cold(backup\_name)

    Create a backup with a filesystem level copy in subdirectory **backup\_name** of
    **$node->backup\_dir**, including WAL. The server must be
    stopped as no attempt to handle concurrent writes is made.

    Use **backup** if you want to back up a running server.

- $node->init\_from\_backup(root\_node, backup\_name, %params)

    Initialize a node from a backup, which may come from this node or a different
    node. root\_node must be a PostgreSQL::Test::Cluster reference, backup\_name the string name
    of a backup previously created on that node with $node->backup.

    Does not start the node after initializing it.

    By default, the backup is assumed to be plain format.  To restore from
    a tar-format backup, pass the name of the tar program to use in the
    keyword parameter tar\_program.

    If there are tablespace present in the backup, include tablespace\_map as
    a keyword parameter whose values is a hash. When combine\_with\_prior is used,
    the hash keys are the tablespace pathnames used in the backup; otherwise,
    they are tablespace OIDs.  In either case, the values are the tablespace
    pathnames that should be used for the target cluster.

    To restore from an incremental backup, pass the parameter combine\_with\_prior
    as a reference to an array of prior backup names with which this backup
    is to be combined using pg\_combinebackup.

    Streaming replication can be enabled on this node by passing the keyword
    parameter has\_streaming => 1. This is disabled by default.

    Restoring WAL segments from archives using restore\_command can be enabled
    by passing the keyword parameter has\_restoring => 1. This is disabled by
    default.

    If has\_restoring is used, standby mode is used by default.  To use
    recovery mode instead, pass the keyword parameter standby => 0.

    The backup is copied, leaving the original unmodified. pg\_hba.conf is
    unconditionally set to enable replication connections.

- $node->rotate\_logfile()

    Switch to a new PostgreSQL log file.  This does not alter any running
    PostgreSQL process.  Subsequent method calls, including pg\_ctl invocations,
    will use the new name.  Return the new name.

- $node->start(%params) => success\_or\_failure

    Wrapper for pg\_ctl start

    Start the node and wait until it is ready to accept connections.

    - fail\_ok => 1

        By default, failure terminates the entire `prove` invocation.  If given,
        instead return a true or false value to indicate success or failure.

- $node->kill9()

    Send SIGKILL (signal 9) to the postmaster.

    Note: if the node is already known stopped, this does nothing.
    However, if we think it's running and it's not, it's important for
    this to fail.  Otherwise, tests might fail to detect server crashes.

- $node->stop(mode)

    Stop the node using pg\_ctl -m $mode and wait for it to stop.

    Note: if the node is already known stopped, this does nothing.
    However, if we think it's running and it's not, it's important for
    this to fail.  Otherwise, tests might fail to detect server crashes.

    With optional extra param fail\_ok => 1, returns 0 for failure
    instead of bailing out.

- $node->reload()

    Reload configuration parameters on the node.

- $node->restart()

    Wrapper for pg\_ctl restart.

    With optional extra param fail\_ok => 1, returns 0 for failure
    instead of bailing out.

- $node->promote()

    Wrapper for pg\_ctl promote

- $node->logrotate()

    Wrapper for pg\_ctl logrotate

- $node->set\_recovery\_mode()

    Place recovery.signal file.

- $node->set\_standby\_mode()

    Place standby.signal file.

- PostgreSQL::Test::Cluster->new(node\_name, %params)

    Build a new object of class `PostgreSQL::Test::Cluster` (or of a subclass, if you have
    one), assigning a free port number.  Remembers the node, to prevent its port
    number from being reused for another node, and to ensure that it gets
    shut down when the test script exits.

    - port => \[1,65535\]

        By default, this function assigns a port number to each node.  Specify this to
        force a particular port number.  The caller is responsible for evaluating
        potential conflicts and privilege requirements.

    - own\_host => 1

        By default, all nodes use the same PGHOST value.  If specified, generate a
        PGHOST specific to this node.  This allows multiple nodes to use the same
        port.

    - install\_path => '/path/to/postgres/installation'

        Using this parameter is it possible to have nodes pointing to different
        installations, for testing different versions together or the same version
        with different build parameters. The provided path must be the parent of the
        installation's 'bin' and 'lib' directories. In the common case where this is
        not provided, Postgres binaries will be found in the caller's PATH.

- get\_free\_port()

    Locate an unprivileged (high) TCP port that's not currently bound to
    anything.  This is used by `new()`, and also by some test cases that need to
    start other, non-Postgres servers.

    Ports assigned to existing PostgreSQL::Test::Cluster objects are automatically
    excluded, even if those servers are not currently running.

    The port number is reserved so that other concurrent test programs will not
    try to use the same port.

    Note: this is not an instance method. As it's not exported it should be
    called from outside the module as `PostgreSQL::Test::Cluster::get_free_port()`.

- $node->teardown\_node()

    Do an immediate stop of the node

    Any optional extra parameter is passed to ->stop.

- $node->clean\_node()

    Remove the base directory of the node if the node has been stopped.

- $node->safe\_psql($dbname, $sql) => stdout

    Invoke **psql** to run **sql** on **dbname** and return its stdout on success.
    Die if the SQL produces an error. Runs with **ON\_ERROR\_STOP** set.

    Takes optional extra params like timeout and timed\_out parameters with the same
    options as psql.

- $node->psql($dbname, $sql, %params) => psql\_retval

    Invoke **psql** to execute **$sql** on **$dbname** and return the return value
    from **psql**, which is run with on\_error\_stop by default so that it will
    stop running sql and return 3 if the passed SQL results in an error.

    As a convenience, if **psql** is called in array context it returns an
    array containing ($retval, $stdout, $stderr).

    psql is invoked in tuples-only unaligned mode with reading of **.psqlrc**
    disabled.  That may be overridden by passing extra psql parameters.

    stdout and stderr are transformed to UNIX line endings if on Windows. Any
    trailing newline is removed.

    Dies on failure to invoke psql but not if psql exits with a nonzero
    return code (unless on\_error\_die specified).

    If psql exits because of a signal, an exception is raised.

    - stdout => \\$stdout

        **stdout**, if given, must be a scalar reference to which standard output is
        written.  If not given, standard output is not redirected and will be printed
        unless **psql** is called in array context, in which case it's captured and
        returned.

    - stderr => \\$stderr

        Same as **stdout** but gets standard error. If the same scalar is passed for
        both **stdout** and **stderr** the results may be interleaved unpredictably.

    - on\_error\_stop => 1

        By default, the **psql** method invokes the **psql** program with ON\_ERROR\_STOP=1
        set, so SQL execution is stopped at the first error and exit code 3 is
        returned.  Set **on\_error\_stop** to 0 to ignore errors instead.

    - on\_error\_die => 0

        By default, this method returns psql's result code. Pass on\_error\_die to
        instead die with an informative message.

    - timeout => 'interval'

        Set a timeout for the psql call as an interval accepted by **IPC::Run::timer**
        (integer seconds is fine).  This method raises an exception on timeout, unless
        the **timed\_out** parameter is also given.

    - timed\_out => \\$timed\_out

        If **timeout** is set and this parameter is given, the scalar it references
        is set to true if the psql call times out.

    - connstr => **value**

        If set, use this as the connection string for the connection to the
        backend.

    - replication => **value**

        If set, add **replication=value** to the conninfo string.
        Passing the literal value `database` results in a logical replication
        connection.

    - extra\_params => \['--single-transaction'\]

        If given, it must be an array reference containing additional parameters to **psql**.

    e.g.

            my ($stdout, $stderr, $timed_out);
            my $cmdret = $node->psql('postgres', 'SELECT pg_sleep(600)',
                    stdout => \$stdout, stderr => \$stderr,
                    timeout => $PostgreSQL::Test::Utils::timeout_default,
                    timed_out => \$timed_out,
                    extra_params => ['--single-transaction'])

    will set $cmdret to undef and $timed\_out to a true value.

            $node->psql('postgres', $sql, on_error_die => 1);

    dies with an informative message if $sql fails.

- $node->background\_psql($dbname, %params) => PostgreSQL::Test::BackgroundPsql instance

    Invoke **psql** on **$dbname** and return a BackgroundPsql object.

    psql is invoked in tuples-only unaligned mode with reading of **.psqlrc**
    disabled.  That may be overridden by passing extra psql parameters.

    Dies on failure to invoke psql, or if psql fails to connect.  Errors occurring
    later are the caller's problem.  psql runs with on\_error\_stop by default so
    that it will stop running sql and return 3 if passed SQL results in an error.

    Be sure to "quit" the returned object when done with it.

    - on\_error\_stop => 1

        By default, the **psql** method invokes the **psql** program with ON\_ERROR\_STOP=1
        set, so SQL execution is stopped at the first error and exit code 3 is
        returned.  Set **on\_error\_stop** to 0 to ignore errors instead.

    - timeout => 'interval'

        Set a timeout for a background psql session. By default, timeout of
        $PostgreSQL::Test::Utils::timeout\_default is set up.

    - replication => **value**

        If set, add **replication=value** to the conninfo string.
        Passing the literal value `database` results in a logical replication
        connection.

    - extra\_params => \['--single-transaction'\]

        If given, it must be an array reference containing additional parameters to **psql**.

- $node->interactive\_psql($dbname, %params) => BackgroundPsql instance

    Invoke **psql** on **$dbname** and return a BackgroundPsql object, which the
    caller may use to send interactive input to **psql**.

    A timeout of $PostgreSQL::Test::Utils::timeout\_default is set up.

    psql is invoked in tuples-only unaligned mode with reading of **.psqlrc**
    disabled.  That may be overridden by passing extra psql parameters.

    Dies on failure to invoke psql, or if psql fails to connect.
    Errors occurring later are the caller's problem.

    Be sure to "quit" the returned object when done with it.

    - extra\_params => \['--single-transaction'\]

        If given, it must be an array reference containing additional parameters to **psql**.

    - history\_file => **path**

        Cause the interactive **psql** session to write its command history to **path**.
        If not given, the history is sent to **/dev/null**.

    This requires IO::Pty in addition to IPC::Run.

- $node->pgbench($opts, $stat, $out, $err, $name, $files, @args)

    Invoke **pgbench**, with parameters and files.

    - $opts

        Options as a string to be split on spaces.

    - $stat

        Expected exit status.

    - $out

        Reference to a regexp list that must match stdout.

    - $err

        Reference to a regexp list that must match stderr.

    - $name

        Name of test for error messages.

    - $files

        Reference to filename/contents dictionary.

    - @args

        Further raw options or arguments.

- $node->connect\_ok($connstr, $test\_name, %params)

    Attempt a connection with a custom connection string.  This is expected
    to succeed.

    - sql => **value**

        If this parameter is set, this query is used for the connection attempt
        instead of the default.

    - expected\_stdout => **value**

        If this regular expression is set, matches it with the output generated.

    - log\_like => \[ qr/required message/ \]
    - log\_unlike => \[ qr/prohibited message/ \]

        See `log_check(...)`.

- $node->connect\_fails($connstr, $test\_name, %params)

    Attempt a connection with a custom connection string.  This is expected
    to fail.

    - expected\_stderr => **value**

        If this regular expression is set, matches it with the output generated.

    - log\_like => \[ qr/required message/ \]
    - log\_unlike => \[ qr/prohibited message/ \]

        See `log_check(...)`.

- $node->poll\_query\_until($dbname, $query \[, $expected \])

    Run **$query** repeatedly, until it returns the **$expected** result
    ('t', or SQL boolean true, by default).
    Continues polling if **psql** returns an error result.
    Times out after $PostgreSQL::Test::Utils::timeout\_default seconds.
    Returns 1 if successful, 0 if timed out.

- $node->poll\_until\_connection($dbname)

    Try to connect repeatedly, until it we succeed.
    Times out after $PostgreSQL::Test::Utils::timeout\_default seconds.
    Returns 1 if successful, 0 if timed out.

- $node->command\_ok(...)

    Runs a shell command like PostgreSQL::Test::Utils::command\_ok, but with PGHOST and PGPORT set
    so that the command will default to connecting to this PostgreSQL::Test::Cluster.

- $node->command\_fails(...)

    PostgreSQL::Test::Utils::command\_fails with our connection parameters. See command\_ok(...)

- $node->command\_like(...)

    PostgreSQL::Test::Utils::command\_like with our connection parameters. See command\_ok(...)

- $node->command\_fails\_like(...)

    PostgreSQL::Test::Utils::command\_fails\_like with our connection parameters. See command\_ok(...)

- $node->command\_checks\_all(...)

    PostgreSQL::Test::Utils::command\_checks\_all with our connection parameters. See
    command\_ok(...)

- $node->issues\_sql\_like(cmd, expected\_sql, test\_name)

    Run a command on the node, then verify that $expected\_sql appears in the
    server log file.

- $node->log\_content()

    Returns the contents of log of the node

- $node->log\_check($offset, $test\_name, %parameters)

    Check contents of server logs.

    - $test\_name

        Name of test for error messages.

    - $offset

        Offset of the log file.

    - log\_like => \[ qr/required message/ \]

        If given, it must be an array reference containing a list of regular
        expressions that must match against the server log, using
        `Test::More::like()`.

    - log\_unlike => \[ qr/prohibited message/ \]

        If given, it must be an array reference containing a list of regular
        expressions that must NOT match against the server log.  They will be
        passed to `Test::More::unlike()`.

- log\_contains(pattern, offset)

    Find pattern in logfile of node after offset byte.

- $node->run\_log(...)

    Runs a shell command like PostgreSQL::Test::Utils::run\_log, but with connection parameters set
    so that the command will default to connecting to this PostgreSQL::Test::Cluster.

- $node->lsn(mode)

    Look up WAL locations on the server:

        * insert location (primary only, error on replica)
        * write location (primary only, error on replica)
        * flush location (primary only, error on replica)
        * receive location (always undef on primary)
        * replay location (always undef on primary)

    mode must be specified.

- $node->wait\_for\_event(wait\_event\_name, backend\_type)

    Poll pg\_stat\_activity until backend\_type reaches wait\_event\_name.

- $node->wait\_for\_catchup(standby\_name, mode, target\_lsn)

    Wait for the replication connection with application\_name standby\_name until
    its 'mode' replication column in pg\_stat\_replication equals or passes the
    specified or default target\_lsn.  By default the replay\_lsn is waited for,
    but 'mode' may be specified to wait for any of sent|write|flush|replay.
    The replication connection must be in a streaming state.

    When doing physical replication, the standby is usually identified by
    passing its PostgreSQL::Test::Cluster instance.  When doing logical
    replication, standby\_name identifies a subscription.

    When not in recovery, the default value of target\_lsn is $node->lsn('write'),
    which ensures that the standby has caught up to what has been committed on
    the primary.

    When in recovery, the default value of target\_lsn is $node->lsn('replay')
    instead which ensures that the cascaded standby has caught up to what has been
    replayed on the standby.

    If you pass an explicit value of target\_lsn, it should almost always be
    the primary's write LSN; so this parameter is seldom needed except when
    querying some intermediate replication node rather than the primary.

    If there is no active replication connection from this peer, waits until
    poll\_query\_until timeout.

    Requires that the 'postgres' db exists and is accessible.

    This is not a test. It die()s on failure.

- $node->wait\_for\_replay\_catchup($standby\_name \[, $base\_node \])

    Wait for the replication connection with application\_name _$standby\_name_
    until its **replay** replication column in pg\_stat\_replication in _$node_
    equals or passes the _$base\_node_'s **replay\_lsn**. If _$base\_node_ is
    omitted, the LSN to wait for is obtained from _$node_.

    The replication connection must be in a streaming state.

    Requires that the 'postgres' db exists and is accessible.

    This is not a test. It die()s on failure.

- $node->wait\_for\_slot\_catchup(slot\_name, mode, target\_lsn)

    Wait for the named replication slot to equal or pass the supplied target\_lsn.
    The location used is the restart\_lsn unless mode is given, in which case it may
    be 'restart' or 'confirmed\_flush'.

    Requires that the 'postgres' db exists and is accessible.

    This is not a test. It die()s on failure.

    If the slot is not active, will time out after poll\_query\_until's timeout.

    target\_lsn may be any arbitrary lsn, but is typically $primary\_node->lsn('insert').

    Note that for logical slots, restart\_lsn is held down by the oldest in-progress tx.

- $node->wait\_for\_subscription\_sync(publisher, subname, dbname)

    Wait for all tables in pg\_subscription\_rel to complete the initial
    synchronization (i.e to be either in 'syncdone' or 'ready' state).

    If the publisher node is given, additionally, check if the subscriber has
    caught up to what has been committed on the primary. This is useful to
    ensure that the initial data synchronization has been completed after
    creating a new subscription.

    If there is no active replication connection from this peer, wait until
    poll\_query\_until timeout.

    This is not a test. It die()s on failure.

- $node->wait\_for\_log(regexp, offset)

    Waits for the contents of the server log file, starting at the given offset, to
    match the supplied regular expression.  Checks the entire log if no offset is
    given.  Times out after $PostgreSQL::Test::Utils::timeout\_default seconds.

    If successful, returns the length of the entire log file, in bytes.

- $node->query\_hash($dbname, $query, @columns)

    Execute $query on $dbname, replacing any appearance of the string \_\_COLUMNS\_\_
    within the query with a comma-separated list of @columns.

    If \_\_COLUMNS\_\_ does not appear in the query, its result columns must EXACTLY
    match the order and number (but not necessarily alias) of supplied @columns.

    The query must return zero or one rows.

    Return a hash-ref representation of the results of the query, with any empty
    or null results as defined keys with an empty-string value. There is no way
    to differentiate between null and empty-string result fields.

    If the query returns zero rows, return a hash with all columns empty. There
    is no way to differentiate between zero rows returned and a row with only
    null columns.

- $node->slot(slot\_name)

    Return hash-ref of replication slot data for the named slot, or a hash-ref with
    all values '' if not found. Does not differentiate between null and empty string
    for fields, no field is ever undef.

    The restart\_lsn and confirmed\_flush\_lsn fields are returned verbatim, and also
    as a 2-list of \[highword, lowword\] integer. Since we rely on Perl 5.14 we can't
    "use bigint", it's from 5.20, and we can't assume we have Math::Bigint from CPAN
    either.

- $node->pg\_recvlogical\_upto(self, dbname, slot\_name, endpos, timeout\_secs, ...)

    Invoke pg\_recvlogical to read from slot\_name on dbname until LSN endpos, which
    corresponds to pg\_recvlogical --endpos.  Gives up after timeout (if nonzero).

    Disallows pg\_recvlogical from internally retrying on error by passing --no-loop.

    Plugin options are passed as additional keyword arguments.

    If called in scalar context, returns stdout, and die()s on timeout or nonzero return.

    If called in array context, returns a tuple of (retval, stdout, stderr, timeout).
    timeout is the IPC::Run::Timeout object whose is\_expired method can be tested
    to check for timeout. retval is undef on timeout.

- $node->corrupt\_page\_checksum(self, file, page\_offset)

    Intentionally corrupt the checksum field of one page in a file.
    The server must be stopped for this to work reliably.

    The file name should be specified relative to the cluster datadir.
    page\_offset had better be a multiple of the cluster's block size.

- $node->log\_standby\_snapshot(self, standby, slot\_name)

    Log a standby snapshot on primary once the slot restart\_lsn is determined on
    the standby.

- $node->create\_logical\_slot\_on\_standby(self, primary, slot\_name, dbname)

    Create logical replication slot on given standby

- $node->validate\_slot\_inactive\_since(self, slot\_name, reference\_time)

    Validate inactive\_since value of a given replication slot against the reference
    time and return it.

- $node->advance\_wal(num)

    Advance WAL of node by given number of segments.
