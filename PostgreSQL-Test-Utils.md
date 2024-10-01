# NAME

PostgreSQL::Test::Utils - helper module for writing PostgreSQL's `prove` tests.

# SYNOPSIS

    use PostgreSQL::Test::Utils;

    # Test basic output of a command
    program_help_ok('initdb');
    program_version_ok('initdb');
    program_options_handling_ok('initdb');

    # Test option combinations
    command_fails(['initdb', '--invalid-option'],
                'command fails with invalid option');
    my $tempdir = PostgreSQL::Test::Utils::tempdir;
    command_ok('initdb', '-D', $tempdir);

    # Miscellanea
    print "on Windows" if $PostgreSQL::Test::Utils::windows_os;
    ok(check_mode_recursive($stream_dir, 0700, 0600),
      "check stream dir permissions");
    PostgreSQL::Test::Utils::system_log('pg_ctl', 'kill', 'QUIT', $slow_pid);

# DESCRIPTION

`PostgreSQL::Test::Utils` contains a set of routines dedicated to environment setup for
a PostgreSQL regression test run and includes some low-level routines
aimed at controlling command execution, logging and test functions.

# EXPORTED VARIABLES

- `$windows_os`

    Set to true when running under Windows, except on Cygwin.

- `$is_msys2`

    Set to true when running under MSYS2.

# ROUTINES

- all\_tests\_passing()

    Return 1 if all the tests run so far have passed. Otherwise, return 0.

- tempdir(prefix)

    Securely create a temporary directory inside `$tmp_check`, like `mkdtemp`,
    and return its name.  The directory will be removed automatically at the
    end of the tests, unless the environment variable PG\_TEST\_NOCLEAN is provided.

    If `prefix` is given, the new directory is templated as `${prefix}_XXXX`.
    Otherwise the template is `tmp_test_XXXX`.

- tempdir\_short()

    As above, but the directory is outside the build tree so that it has a short
    name, to avoid path length issues.

- has\_wal\_read\_bug()

    Returns true if $tmp\_check is subject to a sparc64+ext4 bug that causes WAL
    readers to see zeros if another process simultaneously wrote the same offsets.
    Consult this in tests that fail frequently on affected configurations.  The
    bug has made streaming standbys fail to advance, reporting corrupt WAL.  It
    has made COMMIT PREPARED fail with "could not read two-phase state from WAL".
    Non-WAL PostgreSQL reads haven't been affected, likely because those readers
    and writers have buffering systems in common.  See
    https://postgr.es/m/20220116210241.GC756210@rfd.leadboat.com for details.

- system\_log(@cmd)

    Run (via `system()`) the command passed as argument; the return
    value is passed through.

- system\_or\_bail(@cmd)

    Run (via `system()`) the command passed as argument, and returns
    if the command is successful.
    On failure, abandon further tests and exit the program.

- run\_log(@cmd)

    Run the given command via `IPC::Run::run()`, noting it in the log.
    The return value from the command is passed through.

- run\_command(cmd)

    Run (via `IPC::Run::run()`) the command passed as argument.
    The return value from the command is ignored.
    The return value is `($stdout, $stderr)`.

- pump\_until(proc, timeout, stream, until)

    Pump until string is matched on the specified stream, or timeout occurs.

- generate\_ascii\_string(from\_char, to\_char)

    Generate a string made of the given range of ASCII characters.

- slurp\_dir(dir)

    Return the complete list of entries in the specified directory.

- slurp\_file(filename \[, $offset\])

    Return the full contents of the specified file, beginning from an
    offset position if specified.

- append\_to\_file(filename, str)

    Append a string at the end of a given file.  (Note: no newline is appended at
    end of file.)

- string\_replace\_file(filename, find, replace)

    Find and replace string of a given file.

- check\_mode\_recursive(dir, expected\_dir\_mode, expected\_file\_mode, ignore\_list)

    Check that all file/dir modes in a directory match the expected values,
    ignoring files in `ignore_list` (basename only).

- chmod\_recursive(dir, dir\_mode, file\_mode)

    `chmod` recursively each file and directory within the given directory.

- scan\_server\_header(header\_path, regexp)

    Returns an array that stores all the matches of the given regular expression
    within the PostgreSQL installation's `header_path`.  This can be used to
    retrieve specific value patterns from the installation's header files.

- check\_pg\_config(regexp)

    Return the number of matches of the given regular expression
    within the installation's `pg_config.h`.

- dir\_symlink(oldname, newname)

    Portably create a symlink for a directory. On Windows this creates a junction
    point. Elsewhere it just calls perl's builtin symlink.

# Test::More-LIKE METHODS

- command\_ok(cmd, test\_name)

    Check that the command runs (via `run_log`) successfully.

- command\_fails(cmd, test\_name)

    Check that the command fails (when run via `run_log`).

- command\_exit\_is(cmd, expected, test\_name)

    Check that the command exit code matches the expected exit code.

- program\_help\_ok(cmd)

    Check that the command supports the `--help` option.

- program\_version\_ok(cmd)

    Check that the command supports the `--version` option.

- program\_options\_handling\_ok(cmd)

    Check that a command with an invalid option returns a non-zero
    exit code and error message.

- command\_like(cmd, expected\_stdout, test\_name)

    Check that the command runs successfully and the output
    matches the given regular expression.

- command\_like\_safe(cmd, expected\_stdout, test\_name)

    Check that the command runs successfully and the output
    matches the given regular expression.  Doesn't assume that the
    output files are closed.

- command\_fails\_like(cmd, expected\_stderr, test\_name)

    Check that the command fails and the error message matches
    the given regular expression.

- command\_checks\_all(cmd, ret, out, err, test\_name)

    Run a command and check its status and outputs.
    Arguments:

    - `cmd`: Array reference of command and arguments to run
    - `ret`: Expected exit code
    - `out`: Expected stdout from command
    - `err`: Expected stderr from command
    - `test_name`: test name
