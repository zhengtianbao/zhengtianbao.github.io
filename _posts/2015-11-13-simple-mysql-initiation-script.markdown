---
title: "Simple MySQL Initiation Script"
date: 2015-10-15 15:46:53
categories: ["2015"]
tags: [trove, mysql]
---

一个简单的 MySQL 初始化脚本，用于配置虚机内部的 MySQL 服务，参考了 `OpenStack` `Trove` 项目里的代码。

基本原理:

1. 脚本放置与镜像中，基于该镜像创建的虚机启动的时候会运行该脚本
2. 镜像中 MySQL 服务默认已安装好，同时已授权用户 *os_admin* 拥有超级用户权限
3. MySQL 配置项通过 `nova` 的 [*config-driver*](http://docs.openstack.org/user-guide/cli_config_drive.html) 获取
4. MySQL 的 *datadir* 跑在 `cinder` 创建的 *volume* 中，因此虚机会挂载好一块 *volume*
5. 通过参数 *is_master* 判断是否需要搭建主从，如果是从则需要额外配置

```python
#!/usr/bin/env python

import json
import logging
import os
import signal
import subprocess
import sys
import time

import paramiko
import pexpect


logging.basicConfig(level=logging.DEBUG,
                format='%(asctime)s %(filename)s[line:%(lineno)d] %(levelname)s %(message)s',
                datefmt='%a, %d %b %Y %H:%M:%S',
                filename='/var/log/mysql_replication.log',
                filemode='a')

basedir = ''
datadir = ''
innodb_buffer_pool_size = ''
max_connections = 100
port = 3306
pid_file = ''
log_error = ''
is_master = True

master_host = ''
master_port = 3306
master_user = 'mysql'
master_password = 'mysql1234'

MYSQL_DIR = '/mysql/mysql-5.6.21'
MYSQL_BIN_DIR = '/mysql/mysql-5.6.21/bin'

MASTER_MYSQL_SUPERUSER = 'os_admin'
MASTER_MYSQL_SUPERUSER_PASSWORD = 'EnzwtvvED76fBR4a6u86tRjNsKYtfMTRVApW'
REPLICATION_USER = 'reple'
REPLICATION_PASSWORD = 'replsel0919'
MYCNF_REPLMASTER = "/etc/mysql/conf.d/0replication.cnf"

CONFIG = """
[mysqld]
basedir = %s
datadir = %s
innodb_buffer_pool_size = %s
max_connections = %s
port = %s
pid-file = %s
log-error = %s

sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
!includedir /etc/mysql/conf.d/
"""
MASTER_CONFIG = """
[mysqld]
server-id = 10001
log_bin = %s/mysql-bin.log
"""
SLAVE_CONFIG = """
[mysqld]
server-id = 20001
log_bin = %s/mysql-bin.log
relay_log = %s/mysql-relay-bin.log
read_only = true
slave-skip-errors = 1049
"""
config = ''
overrides = ''
master_config = ''
slave_config = ''

TMP_MYCNF = "/tmp/my.cnf.tmp"
MYSQL_CONFIG = "/etc/my.cnf"
MYCNF_OVERRIDES = "/etc/mysql/conf.d/overrides.cnf"
MYCNF_OVERRIDES_TMP = "/tmp/overrides.cnf.tmp"

_ssh = None

def ssh_connect(host, user, password):
    global _ssh

    _ssh = paramiko.SSHClient()
    _ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    _ssh.connect(host, username=user, password=password)

def _read_paramimko_stream(recv_func):
    result = ''
    buf = recv_func(1024)
    while buf != '':
        result += buf
        buf = recv_func(1024)

    return result

def _execute_remote_command(cmd):
    global _ssh

    chan = _ssh.get_transport().open_session()
    chan.exec_command(cmd)
    stdout = _read_paramimko_stream(chan.recv)

    ret_code = chan.recv_exit_status()

    if ret_code:
        print 'Remote run command Error: %s' % str(cmd)

    return ret_code, stdout



class TimeoutError(Exception):
    pass

class Timeout(object):
    def __init__(self, seconds=1, error_message='Timeout'):
        self.seconds = seconds
        self.error_message = error_message

    def handle_timeout(self, signum, frame):
        raise TimeoutError(self.error_message)

    def __enter__(self):
        signal.signal(signal.SIGALRM, self.handle_timeout)
        signal.alarm(self.seconds)

    def __exit__(self, type, value, traceback):
        signal.alarm(0)

def _execute_command(cmd, timeout=30):
    with Timeout(timeout):
        process = subprocess.Popen(cmd, shell=True,
                            stdin=subprocess.PIPE,
                            stderr=subprocess.PIPE)
        output, error = process.communicate()
        return output, error

def enable_as_master():
    _execute_remote_command('echo "%s" > %s' % 
            (master_config, MYCNF_REPLMASTER))
    _execute_remote_command('/etc/init.d/mysql restart')
    _execute_remote_command("%s -u %s -p%s -e '%s'" % 
            (MYSQL_BIN_DIR + '/mysql', MASTER_MYSQL_SUPERUSER, MASTER_MYSQL_SUPERUSER_PASSWORD, 
             'grant USAGE ON *.* to "%s"@"%%"' % REPLICATION_USER))
    _execute_remote_command("%s -u %s -p%s -e '%s'" % 
            (MYSQL_BIN_DIR + '/mysql', MASTER_MYSQL_SUPERUSER, MASTER_MYSQL_SUPERUSER_PASSWORD, 
             'grant REPLICATION SLAVE ON *.* to "%s"@"%%" IDENTIFIED BY "%s"' % (REPLICATION_USER, REPLICATION_PASSWORD)))

def enable_as_slave():
    _execute_command('echo "%s" > %s' %
            (slave_config, MYCNF_REPLMASTER))
    logging.debug('write slave config'')
    stop_mysql()
    start_mysql()
    logging.debug('restart mysql ')
    change_master_cmd = ("CHANGE MASTER TO MASTER_HOST='%(host)s', "
                         "MASTER_PORT=%(port)s, "
                         "MASTER_USER='%(user)s', "
                         "MASTER_PASSWORD='%(password)s', "
                         "MASTER_HEARTBEAT_PERIOD=10 " %
                         #"MASTER_LOG_FILE='%(log_file)s', "
                         #"MASTER_LOG_POS=%(log_pos)s" %
                         {
                             'host': master_host,
                             'port': master_port,
                             'user': REPLICATION_USER,
                             'password': REPLICATION_PASSWORD,
                             #'log_file': logging_config['log_file'],
                             #'log_pos': logging_config['log_position']
                         })
    _execute_command('%s/mysql -e "%s"' % (MYSQL_BIN_DIR.rstrip('/'), change_master_cmd))
    logging.debug('mysql change master cmd')
    _execute_command('%s/mysql < %s' % (MYSQL_BIN_DIR.rstrip('/'), '/tmp/mysqldump.log'))
    logging.debug('mysqldump cmd')
    _execute_command('%s/mysql -e "start slave"' % (MYSQL_BIN_DIR.rstrip('/')))
    logging.debug('mysql start slave cmd')

def get_snapshot():
    dump_cmd = ("%s -u%s -p%s "
            "--default-character-set=utf8 "
            "--opt --extended-insert=false "
            "--triggers -R --single-transaction "
            "--master-data=1 "
            "--all-databases "
            "> /tmp/mysqldump.log" % ( MYSQL_BIN_DIR + '/mysqldump', MASTER_MYSQL_SUPERUSER, MASTER_MYSQL_SUPERUSER_PASSWORD))

    rcode, rst = _execute_remote_command(dump_cmd)
    if rcode:
        print 'Remote mysqldump error'
    ftp = _ssh.open_sftp()
    try:
        ftp.get('/tmp/mysqldump.log', '/tmp/mysqldump.log')
    except IOError:
        print 'Remote mysqldump.log not exist'
    finally:
        ftp.close()

def get_db_status():
    out, err = _execute_command("%s/mysqladmin ping" % (MYSQL_BIN_DIR.rstrip('/')))
    if err:
        return 'shutdown'
    else:
        return 'running'

def wait_for_real_status_to_change_to(status, max_time):
    WAIT_TIME = 3
    waited_time = 0
    while waited_time < max_time:
        time.sleep(WAIT_TIME)
        waited_time += WAIT_TIME
        actual_status = get_db_status()
        if actual_status == status:
           return True
    return False

def start_mysql():
    try:
        _execute_command("service mysql start")
    except:
        pass
    if not wait_for_real_status_to_change_to('running', 30):
        print "Start up of MySQL failed."

def stop_mysql():
    try:
        _execute_command("service mysql stop")
    except:
        pass
    if not wait_for_real_status_to_change_to('shutdown', 30):
        print "Error stopping MySQL."

def unmount_device(device_path):
    mount_points = []
    try:
        cmd = "grep %s /etc/mtab | awk '{print $2}'" % device_path
        stdout, stderr = _execute_command(cmd)
        mount_points = stdout.strip().split('\n')
    except:
        pass

    for mnt in mount_points:
        if os.path.exists(mnt):
            cmd = "umount %s" % mnt
            child = pexpect.spawn(cmd)
            child.expect(pexpect.EOF)

def format_device(device_path):
    # check device status
    check_cmd = "blockdev --getsize64 %s" % device_path
    stdout, stderr = _execute_command(check_cmd)
    if stderr:
        print 'Error getting device status'
    # format
    cmd = "mkfs -t ext3 -m 5 %s" % device_path
    child = pexpect.spawn(cmd, timeout=180)
    child.expect(pexpect.EOF)
    # check formatted
    cmd = "dumpe2fs %s" % device_path
    child = pexpect.spawn(cmd)
    try:
        i = child.expect(['has_journal', 'Wrong magic number'])
        if i == 0:
            return
        raise
    except pexpect.EOF:
            raise
    child.expect(pexpect.EOF)

def mount_device(device_path, mount_point):
    if not os.path.exists(mount_point):
        _execute_command("mkdir -p %s" % mount_point)
    cmd = "mount %s %s" % (device_path, mount_point)
    child = pexpect.spawn(cmd)
    child.expect(pexpect.EOF)

def update_owner(user, group, path):
    cmd = "chown -R %s:%s %s" % (user, group, path)
    _execute_command(cmd)

def migrate_data(old_datadir, new_datadir):
    cmd = "cp -R %s/* %s" % (old_datadir.rstrip('/'), new_datadir)
    _execute_command(cmd)

def write_cnf(config_contents, overrides):
    try:
        with open(TMP_MYCNF, 'w') as t:
            t.write(config_contents)
            _execute_command("mv %s %s" % (TMP_MYCNF, MYSQL_CONFIG))
    except Exception:
        os.unlink(TMP_MYCNF)
        raise

    if overrides:
        write_config_overrides(overrides)

def write_config_overrides(overrides):
    with open(MYCNF_OVERRIDES_TMP, 'w') as overrides:
        overrides.write(overrideValues)
        _execute_command("mv %s %s" % (MYCNF_OVERRIDES_TMP, MYCNF_OVERRIDES))
        _execute_command("chmod 0644 %s" % MYCNF_OVERRIDES)


def start_master():
    stop_mysql()
    unmount_device('/dev/vdb')
    format_device('/dev/vdb')
    logging.debug('format device successful')
    mount_device('/dev/vdb', '/mydata/')
    logging.debug('mount device successful')
    migrate_data('/mysql/mysql-5.6.21/data/', datadir)
    logging.debug('migrate data successful')
    update_owner('mysql', 'mysql', datadir)
    write_cnf(config, overrides)
    logging.debug('write cnf successful')
    start_mysql()
    logging.debug('start master successful')


def start_slave():
    start_master()
    ssh_connect(master_host, master_user, master_password)
    logging.debug('ssh connect successful')
    enable_as_master()
    logging.debug('remote enable as master successful')
    get_snapshot()
    logging.debug('get snapshot successful')
    enable_as_slave()
    logging.debug('enable as slaver successful')


def read_config(faker=False):
    if faker:
        meta = {'basedir': '/mysql/mysql-5.6.21',
                  'datadir': '/mydata/',
                  'innodb_buffer_pool_size': '4G',
                  'max_connections': 800,
                  'master_ip': '',
                  'master_port': 3306,
                  'master_machine_root_password': 'KeyTone123',
                  'pid-file': '/mydata/ucoasdb.pid',
                  'log-error': '/mydata/ucoasdb.err',
                  'is_master': True
                }
    else:
        config_driver = '/dev/disk/by-label/config-2'
        if not os.path.exists(config_driver):
            logging.warning('config driver file does not exist')
            raise
        mount_point = '/mnt/config'
        mount_device(config_driver, mount_point)
        with open('/mnt/config/openstack/latest/meta_data.json', 'r') as f:
            json_config = f.read()
        unmount_device('/dev/disk/by-label/config-2')
        dict_config = json.loads(json_config)
        meta = dict_config['meta']
        logging.debug('config meta: %s' % str(meta))

    global basedir, datadir, innodb_buffer_pool_size, max_connections, port, \
           pid_file, log_error, is_master, config, overrides, master_config, \
           slave_config, master_host, master_port, master_user, master_password

    basedir = meta.get('basedir', '/mysql/mysql-5.6.21')
    datadir = meta.get('datadir', '/mydata/')
    innodb_buffer_pool_size = meta.get('innodb_buffer_pool_size', '4G')
    max_connections = meta.get('max_connections', 800)
    port = meta.get('port', 3306)
    pid_file = meta.get('pid-file', '/mydata/ucoasdb.pid')
    log_error = meta.get('log-error', '/mydata/ucoasdb.err')
    is_master = meta.get('is_master', True)
    is_master = True if is_master in ('True', 'true', True) else False

    config = CONFIG % (basedir, datadir, innodb_buffer_pool_size, 
                   max_connections, port, pid_file, log_error)
    overrides = None
    master_config = MASTER_CONFIG % datadir.rstrip('/')
    slave_config = SLAVE_CONFIG % (datadir.rstrip('/'), datadir.rstrip('/'))

    master_host = meta.get('master_ip')
    master_port = meta.get('master_port')
    master_user = 'mysql'
    master_password = 'mysql1234'


def delete_me():
    _execute_command("mv %s %s" % ('/root/mysql_replication.py', '/root/.mysql_replication.py'))

if __name__ == '__main__':
    try:
        read_config(faker=False)
    except:
        logging.warning('Read config file failed.')
        sys.exit()

    if is_master:
        logging.debug('start as master')
        start_master()
    else:
        logging.debug('start as slaver')
        start_slave()
    
    delete_me()
```
