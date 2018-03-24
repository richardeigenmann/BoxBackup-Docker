# docker build -t richardeigenmann/boxbackupdaemon-x64 .
# docker run -d --restart=always --hostname boxbackup -v /usb-disk/boxbackup-data/:/boxbackup-data/ -p 2201:2201 --rm=false richardeigenmann/boxbackupdaemon-x64

FROM opensuse:latest

COPY bbstored.conf /etc/boxbackup/
COPY raidfile.conf /etc/boxbackup/
COPY bbstored /usr/bin/

VOLUME /boxbackup-data/

RUN useradd boxbackup

RUN zypper --non-interactive in db

#Options:
#  -c <file>  Use the specified configuration file. If -c is omitted, the last
#             argument is the configuration file, or else the default
#             [/etc/boxbackup/bbstored.conf]
#  -D         Debugging mode, do not fork, one process only, one client only
#  -F         Do not fork into background, but fork to serve multiple clients
#  -k         Keep console open after fork, keep writing log messages to it
#  -K         Stop writing log messages to console while daemon is running
#  -o <file>  Log to a file, defaults to maximum verbosity
#  -O <level> Set file log verbosity to error/warning/notice/info/trace/everything
#  -L <file>  Filter out log messages except from specified file, can repeat
#             (for example, -L Logging.cpp)
#  -N         Truncate log file at startup and on backup start
#  -P         Show process ID (PID) in console output
#  -q         Run more quietly, reduce verbosity level by one, can repeat
#  -Q         Run at minimum verbosity, log nothing to console and system
#  -t <tag>   Tag console output with specified marker
#  -T         Timestamp console output
#  -U         Timestamp console output with microseconds
#  -v         Run more verbosely, increase verbosity level by one, can repeat
#  -V         Run at maximum verbosity, log everything to console and system
#  -W <level> Set verbosity to error/warning/notice/info/trace/everything

ENTRYPOINT ["/usr/bin/bbstored", "-F", "-T", "-O", "notice", "-o", "/boxbackup-data/logfile"]
#ENTRYPOINT ["bash"]
