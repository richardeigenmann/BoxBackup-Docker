# docker build -t richardeigenmann/boxbackup-x64 .
# docker run -it -hostname boxbackup richardeigenmann/boxbackup-x64
# docker exec -it -hostname boxbackup richardeigenmann/boxbackup-x64 bash
# docker run -d --restart=always --hostname boxbackup -v /usb-disk/boxbackup-data/:/boxbackup-data/ -p 2201:2201 --rm=false richardeigenmann/boxbackup-x64

FROM amd64/gcc:10 AS builder
RUN git clone https://github.com/boxbackup/boxbackup.git
RUN apt-get update && apt-get -y install xsltproc
RUN cd /boxbackup && ./bootstrap
RUN cd /boxbackup && ./configure
RUN cd /boxbackup && make -j$(nproc)

FROM amd64/debian:stable-slim AS bbtempserver
RUN useradd boxbackup
RUN apt-get update && apt-get -y install openssl
COPY raidfile.conf /etc/boxbackup/
COPY --from=builder /boxbackup/release/bin/bbstoreaccounts/bbstoreaccounts /usr/bin
COPY --from=builder /boxbackup/bin/bbstored/bbstored-config /usr/bin
COPY --from=builder /boxbackup/bin/bbstored/bbstored-certs /usr/bin
# Client programs
COPY --from=builder /boxbackup/release/bin/bbackupd/bbackupd /usr/bin
COPY --from=builder /boxbackup/release/bin/bbackupctl/bbackupctl /usr/bin
COPY --from=builder /boxbackup/release/bin/bbackupquery/bbackupquery /usr/bin
COPY --from=builder /boxbackup/bin/bbackupd/bbackupd-config /usr/bin
RUN apt-get update && apt-get -y install vim less

FROM amd64/debian:stable-slim AS bbserver
RUN useradd boxbackup
RUN apt-get update && apt-get -y install openssl libstdc++6
COPY bbstored.conf /etc/boxbackup/
COPY raidfile.conf /etc/boxbackup/
COPY --from=builder /boxbackup/release/bin/bbstored/bbstored /usr/bin
COPY --from=builder /boxbackup/release/bin/bbstoreaccounts/bbstoreaccounts /usr/bin
COPY --from=builder /boxbackup/bin/bbstored/bbstored-config /usr/bin

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
