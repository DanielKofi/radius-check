! usr/bin/python

from datetime import timedelta, datetime,date
import paramiko
import re
import time
import sys
import threading
import getpass
import string

#username / password authentication
def getCred():
    un = getpass.getpass("Username : ")
    pw = getpass.getpass("Password : ")
    return[un,pw]


# Prompt user for date and validate   
def dateCheck():
    rdate = input('enter date should be YYYYMMDD: ')
    try:
        datetime.strptime(rdate, '%Y%m%d')
    except:
        print("Incorrect data format")
        dateCheck()
    return rdate

# Prompt user for mac address and validate
def macCheck():
    macAdd = input('enter mac address xx-xx-xx-xx-xx-xx : ')
    pat = re.compile('[0-9a-f]{2}([-])[0-9a-f]{2}(\\1[0-9a-f]{2}){4}$')
    if pat.match(macAdd):
        hallsMac = hallsMac = re.sub('[:-]', '', macAdd).upper()
        print('halls mac is :',hallsMac)
        print('mac address is :',macAdd)
    else:
        print('please enter the correct format, i.e should be 2c-33-7a-f9-de-97')
        macAdd()
    return [macAdd,hallsMac]

# Function to check the next date in log from the initial infrigement date
def radCheck(dateForm):
    infLogDt = 'radius.log-' + dateForm + '.gz'
    print(infLogDt)
    t=time.strptime(dateForm,'%Y%m%d')
    nextDate =date(t.tm_year,t.tm_mon,t.tm_mday)+timedelta(+1)
    yDate =date(t.tm_year,t.tm_mon,t.tm_mday)+timedelta(-1)
    tmDate = 'radius.log-' + str(nextDate.strftime('%Y%m%d')) + '.gz'
    print('Checking log file -  ',tmDate)
    return tmDate

def sshRad(ip,user,passw,date1,tmDate1):
    global output
    #date1[0] represents standard mac address
    #date1[1] mac address notation for a Halls connection
    #tmDate is the name of the logfile to be checked
    cmd1 = ('cd /var/log/radius; zgrep ' + date1[0] + ' ' + tmDate1)
    cmd2 = ('cd /var/log/radius; zgrep ' + date1[1] + ' ' + tmDate1)
    try:
        ssh = paramiko.SSHClient()
        ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        ssh.connect(ip, port=22, username=user,
                    password=passw,allow_agent=False,look_for_keys=False)
    except paramiko.AuthenticationException as error:
        print('could not log you onto radius servers, please check that you earlier entered you correct username and password')
        main()


    #send first first search command i.e **zgrep 2c-33-61-ac-dc-e2 radius.log-20170413.gz**
    stdin, stdout, stderr = ssh.exec_command(cmd1)
    output = stdout.readlines()
    if output:
        #print('first output ',output)
        ssh.close()
        printOut()
        
    else:
        # send 2nd command **zgrep 2C3361ACDCE2 radius.log-20170413.gz** with halls mac address
        # notation
        stdin, stdout, stderr = ssh.exec_command(cmd2)
        output = stdout.readlines()
        if output:

            
            print('second output -- ',output)
            ssh.close()
            printOut()
        else:
            print('No data found on ' + ip)

def printOut():
    #print('print out function')
    #print(output)
    for line in output:
        print(line)
        uName = re.findall('\w+@\S+',line)
        #print(uName)
        print('Username of client is '+ str(uName))
        time.sleep(2)
       


def main():
    ip = ['128.40.155.170', '128.40.155.171','128.40.155.178','128.40.155.169','128.40.155.177']
    # 1. Promt user for credentials and returns username and password variables in 'creds[0]' and 'creds[1]' respectively
    creds = getCred()
    # 2. Promt user for date and returns date variable 'dateForm' 
    dateForm = dateCheck()
    # 3. Promts user for mac address and returns 2 different mac address notations 
    date1 = macCheck()
    # 4. Returns log file date 
    tmDate1 = radCheck(dateForm)

    # 5. Theading function to to log onto each radius server  
    threads = []
    for h in ip:   
        t = threading.Thread(target=sshRad, args=(h,creds[0],creds[1],date1,tmDate1))
        time.sleep(2)
        t.start()
        threads.append(t)
    for t in threads:
        t.join()
    
  

main()