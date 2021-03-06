# PythonScripts
Python scripts


import boto3
import logging
import socket
import ssl
from datetime import date
from dateutil import parser
from urllib.error import URLError, HTTPError
from urllib.request import Request
from urllib.request import urlopen
logger = logging.getLogger()
region = 'us-west-2'
old_msg = 'Please be advised:\n\n'
def get_sites():
    try:
        resp = urlopen(Request('https://s3-us-west-2.amazonaws.com/Scripts/urls.txt'))
        return resp.read().decode().split('\n')
    except (HTTPError, URLError) as err:
        logger.error(err)
def check_cert(msg):
    for site in get_sites():
        try:
            ctx = ssl.create_default_context()
            s = ctx.wrap_socket(socket.socket(), server_hostname=site)
            s.connect((site, 443))
            cert = s.getpeercert()
            exp_date = parser.parse(cert.get('notAfter')).date()
            now_date = date.today()
            td = exp_date - now_date
            if 0 < td.days < 90:
                msg += f"{site.replace('www.', '*.')} ~ cert not valid after {exp_date}\n  expiration within {td.days} days\n"
            elif td.days < 0:
                msg += f"{site.replace('www.', '*.')} ~ cert not valid after {exp_date} ~ already expired\n"
        except Exception as err:
            logger.error(err)
    return msg
def lambda_handler(event, context):
    new_msg = check_cert(old_msg)
    if new_msg > old_msg:
        sns_client = boto3.client('sns', region)
        sns_response = sns_client.publish(
            TargetArn='arn:aws:sns:us-west-2:SDSSDSDSD0:DevOps', Message=new_msg,
            Subject='Cert Check Notification - Test', MessageStructure='string',
            MessageAttributes={'string': {'DataType': 'String', 'StringValue': 'message', }}
        )
