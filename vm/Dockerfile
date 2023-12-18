FROM ubuntu:23.10

RUN apt update && apt install -y openssh-server && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash ansible
RUN echo "ansible:admin123" | chpasswd

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]