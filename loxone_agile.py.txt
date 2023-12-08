# The information about how to structure the API Commands can be found here:
# https://www.loxone.com/dede/wp-content/uploads/sites/2/2021/10/API-Commands.pdf
#gck 4 December 2023 - initial version


import requests
from requests.auth import HTTPBasicAuth
from datetime import date
from datetime import datetime
from pytz import timezone
import pytz
import time
import collections

############# i) setup time data

format = "%Y-%m-%d %H:%M:%S %Z%z"
date_format = "%Y-%m-%dT%H:%M:%SZ"

now_utc = datetime.now(timezone('UTC'))

print('Current Time in UTC TimeZone:',now_utc.strftime(format))

day =  datetime.now().day
hour = datetime.now().hour
minute = datetime.now().minute
day =  datetime.now(pytz.timezone('Europe/London')        ).day

print( day, hour, minute )


############# ii) get the octopus data
url = "https://api.octopus.energy/v1/products/AGILE-FLEX-22-11-25/electricity-tariffs/E-1R-AGILE-FLEX-22-11-25-J/standard-unit-rates/"

headers = {"content-type": "application/json"}
r = requests.get(
        f"{url}/",
        headers=headers,
)
results = r.json()["results"]

date_rates = collections.OrderedDict()

for result in results:
            price = result["value_inc_vat"]
            valid_from = result["valid_from"]
            date_rates[valid_from] = price

#print( "results", results )
#print( "date_rates", date_rates )

for row in date_rates:
    print( 'record: ', row, row[8:10] , row[11:16] , row[11:13], row[14:16] )

data = date_rates
data2 = data


############# iii) setup the Loxone call

local_url= "http://192.168.0.XX/"
serial_number = "XXXXXXXXX"

username = "admin"
password = "XXXXXXXXXX"

agile_vti = "agile"
agile_now_vti ="agile_now"

basic = HTTPBasicAuth(username,password)

#external_url = f"http://dns.loxonecloud.com/{serial_number}"
#resolved_external_url = (requests.get(external_url)).url

command_base = f"dev/sps/io/{agile_vti}/"

def execute_local_command(command):
    command = local_url + command_base + command
    return print(requests.get(command,auth =basic))

def execute_value_command(command):
    command_base = "dev/sps/io/agile_now/"
    command = local_url + command_base + command
    print( command )
    return print(requests.get(command,auth =basic))


############# iv) process the Agile data

for row in date_rates:
    date_obj = datetime.strptime(row, date_format)
    date_obj = date_obj.replace(tzinfo=timezone('UTC'))
    timedelta  = now_utc - date_obj
    print(  timedelta.seconds )
    if  timedelta.days == 0  and timedelta.seconds < 1800 :
        print( "!!!!!!!!!!!!!!!now record!", row )
        execute_value_command( str( data[row])  )


#    print( timedelta )
#    print( 'record: ', day, row, row[8:10] , row[11:16] , row[11:13], row[14:16]                     )
    if "00" == row[14:16]:
#        hour = row[11:13]
#        print( "found an hour!!!!" , hour, day )
        date_obj = datetime.strptime(row, date_format)
        date_obj = date_obj.replace(tzinfo=timezone('UTC'))
        timedelta  =date_obj - now_utc
#        print(  timedelta.seconds )
        print("timedelta",  timedelta, timedelta.days, timedelta.seconds//3600)
        print( "found 00 record:", row ,"timedelta.seconds",  timedelta.seconds )
        if ( timedelta.days == 0 ) or ( timedelta.days == -1 and timedelta.seconds > 82800 ):
#            print(  timedelta.seconds )
#        if timedelta.seconds <= 86400 and timedelta.seconds >= 0 :
            for row2 in data2:
                if  row[8:10] == row2[8:10] and "30" == row2[14:16] and row[11:13] == row2[11:13]:
                    print( "found 30 record:", row2 )
                    other_value = data[row2]


            print( "to be published", other_value, ":", data[row])
            avg_value =  round( ( other_value + data[row]  ) / 2 , 3)
#            print( "avg_value", avg_value )
            input_var = "+" + str(timedelta.seconds//3600  + 1)
            if input_var == "+24":
                input_var = "+0"
#            print( "input_var", input_var)
            command = "SET(PO;" + input_var + ";" +  str(avg_value) + ")"
            print("command", command)
            execute_local_command(command)
