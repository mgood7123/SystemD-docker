# Docker System D image

# copy repo

once a systemd image is running, you can copy your github repo into the image via
```yaml
sudo docker cp $GITHUB_WORKSPACE git_local_ubuntu:/home/ubuntu/git_repo

# the repo must be chown as the correct docker user
#
sudo docker container exec --user ubuntu --workdir /home/ubuntu git_local_ubuntu bash -c "DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/\$(id -u ubuntu)/bus systemd-run --user --scope --collect /bin/bash -c \"sudo chown -R ubuntu:ubuntu git_repo\""
```

# executing commands

once a systemd image is running, execute commands inside it via the following

```sh
sudo docker container exec --user ubuntu --workdir /home/ubuntu git_local_ubuntu bash -c "DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/\$(id -u ubuntu)/bus systemd-run --user --scope --collect /bin/bash -c \"<YOUR_COMMAND(s)_HERE>\""
```

note your commands must be double quote escaped

```sh
"\"<cmd>\""

# example, execute cmake inside docker image
# command:                  cmake -G "Unix Makefiles"
# command (double escaped): cmake -G \\\"Unix Makefiles\\\"
DOCKER_EXEC_COMMAND " SYSTEMD_EXEC_COMMAND \" <command (double escaped)> cmake -G \\\"Unix Makefiles\\\" \""
```

# installation

copy and paste the following under a `ubuntu-latest` github runner

```yaml
      - name: download Docker image - 'Ubuntu SystemD'
        run: |
          <download tool> <URL to release ubuntu_systemd.tar archive>

      - name: load docker image
        run: |
          sudo docker image load -i ./ubuntu_systemd.tar

      # THIS ONLY NEED TO BE DONE ONCE
      #
      - name: boot systemd - host remount cgroup
        run: |
          sudo mount -o remount,rw,nosuid,nodev,noexec,relatime /sys/fs/cgroup
```

# booting

once systemd is installed, it must be booted

```yaml
      - name: boot systemd - create systemd container
        run: |
          sudo docker container create --user root --workdir /home/root --tmpfs /tmp --tty --cap-add SYS_ADMIN --cap-add NET_ADMIN --cgroup-parent docker.slice --security-opt apparmor:unconfined --security-opt seccomp=unconfined --ulimit nofile=262144:262144 -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cgroupns host --name git_local_ubuntu git_local_ubuntu:23.10 /sbin/init
      
      - name: boot systemd - start systemd container
        run: |
          sudo docker container start git_local_ubuntu
          sudo docker container exec git_local_ubuntu bash -c "sleep 5 ; systemctl --wait is-system-running || sysctl --failed"
      
      - name: boot systemd - verify
        run: |
          sudo docker container exec git_local_ubuntu bash -c "systemctl status"
      
      - name: boot systemd - create user session
        run: |
          sudo docker container exec git_local_ubuntu bash -c "mkdir -p /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "chown ubuntu:ubuntu /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "systemctl start user@\$(id -u ubuntu)"
```

# rebooting

if for any reason you must reboot the container (eg, you install a new package that uses systemd and it needs a reboot to complete)

```yaml
      - name: reboot systemd - commit image
        run: |
          sudo docker container commit git_local_ubuntu git_local_ubuntu:23.10

      - name: reboot systemd - stop container
        run: |
          sudo docker container stop git_local_ubuntu

      - name: reboot systemd - remove container
        run: |
          sudo docker container rm git_local_ubuntu

      # add any needed commands here, eg flatpak requires --privileged (and we dont know how to get around it)
      - name: reboot systemd - create systemd container
        run: |
          sudo docker container create --user root --workdir /home/root --tmpfs /tmp --tty --cap-add SYS_ADMIN --cap-add NET_ADMIN --cgroup-parent docker.slice --security-opt apparmor:unconfined --security-opt seccomp=unconfined --ulimit nofile=262144:262144 -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cgroupns host --name git_local_ubuntu git_local_ubuntu:23.10 /sbin/init
      
      - name: reboot systemd - start systemd container
        run: |
          sudo docker container start git_local_ubuntu
          sudo docker container exec git_local_ubuntu bash -c "sleep 5 ; systemctl --wait is-system-running || sysctl --failed"
      
      - name: reboot systemd - verify
        run: |
          sudo docker container exec git_local_ubuntu bash -c "systemctl status"
      
      - name: reboot systemd - create user session
        run: |
          sudo docker container exec git_local_ubuntu bash -c "mkdir -p /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "chown ubuntu:ubuntu /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "systemctl start user@\$(id -u ubuntu)"
```

# full example

for this example we shall install `flatpak`

`flatpak` requires a systemd reboot in order to complete its installation

```yaml
      - name: download Docker image - 'Ubuntu SystemD'
        run: |
          <download tool> <URL to release ubuntu_systemd.tar archive>

      - name: load docker image
        run: |
          sudo docker image load -i ./ubuntu_systemd.tar

      # THIS ONLY NEED TO BE DONE ONCE
      #
      - name: boot systemd - host remount cgroup
        run: |
          sudo mount -o remount,rw,nosuid,nodev,noexec,relatime /sys/fs/cgroup

      - name: boot systemd - create systemd container
        run: |
          sudo docker container create --user root --workdir /home/root --tmpfs /tmp --tty --cap-add SYS_ADMIN --cap-add NET_ADMIN --cgroup-parent docker.slice --security-opt apparmor:unconfined --security-opt seccomp=unconfined --ulimit nofile=262144:262144 -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cgroupns host --name git_local_ubuntu git_local_ubuntu:23.10 /sbin/init
      
      - name: boot systemd - start systemd container
        run: |
          sudo docker container start git_local_ubuntu
          sudo docker container exec git_local_ubuntu bash -c "sleep 5 ; systemctl --wait is-system-running || sysctl --failed"
      
      - name: boot systemd - verify
        run: |
          sudo docker container exec git_local_ubuntu bash -c "systemctl status"
      
      - name: boot systemd - create user session
        run: |
          sudo docker container exec git_local_ubuntu bash -c "mkdir -p /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "chown ubuntu:ubuntu /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "systemctl start user@\$(id -u ubuntu)"

      - name: SYSTEMD - install flatpak
        run: |
          sudo docker container exec --user ubuntu --workdir /home/ubuntu git_local_ubuntu bash -c "DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/\$(id -u ubuntu)/bus systemd-run --user --scope --collect /bin/bash -c \"sudo apt install -y flatpak flatpak-builder\""

      - name: reboot systemd - commit image
        run: |
          sudo docker container commit git_local_ubuntu git_local_ubuntu:23.10

      - name: reboot systemd - stop container
        run: |
          sudo docker container stop git_local_ubuntu

      - name: reboot systemd - remove container
        run: |
          sudo docker container rm git_local_ubuntu

      # flatpak requires --privileged and we dont know how to get around it
      - name: reboot systemd - create systemd container
        run: |
          sudo docker container create --user root --workdir /home/root --tmpfs /tmp --tty --cap-add SYS_ADMIN --cap-add NET_ADMIN --privileged --cgroup-parent docker.slice --security-opt apparmor:unconfined --security-opt seccomp=unconfined --ulimit nofile=262144:262144 -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cgroupns host --name git_local_ubuntu git_local_ubuntu:23.10 /sbin/init

      - name: reboot systemd - start systemd container
        run: |
          sudo docker container start git_local_ubuntu
          sudo docker container exec git_local_ubuntu bash -c "sleep 5 ; systemctl --wait is-system-running || sysctl --failed"

      - name: reboot systemd - verify
        run: |
          sudo docker container exec git_local_ubuntu bash -c "systemctl status"

      - name: reboot systemd - create user session
        run: |
          sudo docker container exec git_local_ubuntu bash -c "mkdir -p /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "chown ubuntu:ubuntu /run/user/\$(id -u ubuntu)"
          sudo docker container exec git_local_ubuntu bash -c "systemctl start user@\$(id -u ubuntu)"

      - name: SYSTEMD - add flatpak repository
        run: |
          sudo docker container exec --user ubuntu --workdir /home/ubuntu git_local_ubuntu bash -c "DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/\$(id -u ubuntu)/bus systemd-run --user --scope --collect /bin/bash -c \"sudo flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo\""

      - name: SYSTEMD - install flatpak sdk
        run: |
          sudo docker container exec --user ubuntu --workdir /home/ubuntu git_local_ubuntu bash -c "DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/\$(id -u ubuntu)/bus systemd-run --user --scope --collect /bin/bash -c \"sudo flatpak install -y flathub org.freedesktop.Platform//23.08 org.freedesktop.Sdk//23.08\""
```
