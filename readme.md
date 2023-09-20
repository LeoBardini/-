/*
import os
import re
import sys
import time
import pathlib

run_env_tag = None


def check_run_env():
    global run_env_tag
    result = os.popen(r'adb shell \'echo test\'').read()
    if result.startswith('test'):
        run_env_tag = 'linux'
    else:
        run_env_tag = 'windows'
    print(run_env_tag)


def get_pid_str(cmdline):
    global run_env_tag
    if run_env_tag == 'linux':
        return r'\'`pidof ' + cmdline + r'`\''
    else:
        return r'`pidof ' + cmdline + r'`'


def file_to_string(filepath):
    return pathlib.Path(filepath).read_text(errors='ignore')


def get_current_path():
    return pathlib.Path(__file__).parent.absolute()


def mkdir(path):
    is_exists = os.path.exists(path)
    if not is_exists:
        os.makedirs(path)
        print(path + ' 创建成功')
        return True
    else:
        print(path + ' 目录已存在')
        return False


def get_file_timed_name(end_tag):
    return time.strftime("%Y%m%d_%H%M%S", time.localtime()) + "_" + end_tag


def dump_process_meminfo(process_name, out_dir):
    if len(process_name) > 13:
        file_name = get_file_timed_name(process_name[-13:] + "_meminfo.txt")
    else:
        file_name = get_file_timed_name(process_name + "_meminfo.txt")
    file_path = os.path.join(out_dir, file_name)
    print("dump_process_meminfo")
    if not os.system(r'adb shell dumpsys meminfo ' + get_pid_str(process_name) + " > " + file_path):
        os.system(r'adb shell cat /proc/' + get_pid_str(process_name) + '/status >> ' + file_path)
        os.system(r'adb shell cat /d/mali0/gpu_memory >> ' + file_path)
        os.system(r'adb shell top -s 6 -n 1 >> ' + file_path)
        return file_path
    else:
        return None


# adb shell am dumpheap -g <PID> /data/local/tmp/heap.hprof
def dump_vm_heap(process_name, file_name):
    os.system(r'adb shell am dumpheap -g ' + get_pid_str(process_name) + r' /data/local/tmp/' + file_name)


"""
                   Pss  Private  Private  SwapPss      Rss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty    Total     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------   ------
"""


def get_vm_heap_acllo_from_meminfofile(meminfo_file):
    meminfo = file_to_string(meminfo_file)
    vm_heap = re.search(r'\s*Dalvik Heap\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+(\d+)\s+\d+', meminfo, re.S)
    if vm_heap:
        return int(vm_heap.group(1))
    else:
        print("Meminfo error,Maybe Process not alive!, Or Please check process name is right?")
        return 0


if __name__ == '__main__':
    check_run_env()

    if len(sys.argv) != 4:
        print("命令错误")
        print('输入格式例如:python obvmem.py com.android.settings 5 100')
        print('<com.android.settings>: 进程名')
        print('<5>：                   5分钟检测一次VM heap')
        print('<100>:                  100M时开始dumpheap')
        sys.exit(-1)
    else:
        target_process = sys.argv[1]
    meminfo_dir = os.path.join(get_current_path(), r'meminfo')
    mkdir(meminfo_dir)
    if os.system(r'adb root'):
        print('请确认adb是否可连接，以及是否有root权限')
        sys.exit(-1)
    meminfo_file = dump_process_meminfo(target_process, meminfo_dir)
    if meminfo_file:
        vm_alloc_size = get_vm_heap_acllo_from_meminfofile(meminfo_file)
        if vm_alloc_size > 0:
            hprof_name = get_file_timed_name(target_process + ".hprof")
            print("Alloc size:" + str(vm_alloc_size) + "KB")
            time.sleep(2)
            dump_vm_heap(target_process, hprof_name)
        print("Start Observer")
    while True:
        check_delay_time = 60 * int(sys.argv[2])
        print("Check again after " + str(check_delay_time) + "S")
        time.sleep(check_delay_time)
        meminfo_file = dump_process_meminfo(target_process, meminfo_dir)
        time.sleep(2)
        if meminfo_file:
            vm_alloc_size = get_vm_heap_acllo_from_meminfofile(meminfo_file)
            if vm_alloc_size > (int(sys.argv[3]) * 1024):
                hprof_name = get_file_timed_name(target_process + ".hprof")
                dump_vm_heap(target_process, hprof_name)
            else:
                print("Alloc size:" + str(vm_alloc_size) + "KB, Not dumpheap!")

*/
python obvmem.py xxx.xxx.xx 5 150

xxx.xxx.xx 应用包名
5分钟检查一次
Alloc size 到150M时dump javaheap (如果应用java heap上限是512M，建议设450)

原理：
5分钟获取一次内存信息，保存脚本所在目录meminfo文件夹，当前VM heap alloc超过100M时会dump hprof 到手机/data/local/tmp/文件夹

测试一段时间，确认下/data/local/tmp/中是否有*.prof文件
