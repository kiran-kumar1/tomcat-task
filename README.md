# tomcat-task
---
- name: Install Java 1.7
  yum: name=java-1.7.0-openjdk state=present

- name: add group "tomcat"
  group: name=tomcat

- name: add user "tomcat"
  user: name=tomcat group=tomcat home=/usr/share/tomcat createhome=no
  become: True
  become_method: sudo

- name: Download Tomcat
  get_url: url=http://archive.apache.org/dist/tomcat/tomcat-7/v7.0.61/bin/apache-tomcat-7.0.61.tar.gz dest=/opt/apache-tomcat-7.0.61.tar.gz

- name: Extract archive
  command: chdir=/usr/share /bin/tar xvf /opt/apache-tomcat-7.0.61.tar.gz -C /opt/ creates=/opt/apache-tomcat-7.0.61

- name: Symlink install directory
  file: src=/opt/apache-tomcat-7.0.61 path=/usr/share/tomcat state=link

- name: Change ownership of Tomcat installation
  file: path=/usr/share/tomcat/ owner=tomcat group=tomcat state=directory recurse=yes

- name: Configure Tomcat server
  template: src=server.xml dest=/usr/share/tomcat/conf/
  notify: restart tomcat

- name: Configure Tomcat users
  template: src=tomcat-users.xml dest=/usr/share/tomcat/conf/
  notify: restart tomcat

- name: Install Tomcat init script
  copy: src=tomcat-initscript.sh dest=/etc/init.d/tomcat mode=0755

- name: Start Tomcat
  service: name=tomcat state=started enabled=yes

- name: deploy iptables rules
  template: src=iptables-save dest=/etc/sysconfig/iptables
  when: "ansible_os_family == 'RedHat' and ansible_distribution_major_version == '6'"
  notify: restart iptables

- name: insert firewalld rule for tomcat http port
  firewalld: port={{ http_port }}/tcp permanent=true state=enabled immediate=yes
  when: "ansible_os_family == 'RedHat' and ansible_distribution_major_version == '7'"

- name: insert firewalld rule for tomcat https port
  firewalld: port={{ https_port }}/tcp permanent=true state=enabled immediate=yes
  when: "ansible_os_family == 'RedHat' and ansible_distribution_major_version == '7'"

- name: wait for tomcat to start
  wait_for: port={{http_port}}
