# docker build -t richardeigenmann/bbstoreaccounts-arm .
# docker run -it  --hostname boxbackup -v /usb-disk/boxbackup-data/:/boxbackup-data/  richardeigenmann/bbstoreaccounts-arm


FROM resin/rpi-raspbian
COPY bbstored.conf /etc/boxbackup/
COPY raidfile.conf /etc/boxbackup/
COPY bbstoreaccounts /usr/bin/

VOLUME /boxbackup-data/

RUN useradd boxbackup

ENTRYPOINT ["bash"]
