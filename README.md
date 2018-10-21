# data_collection
This script gathers data from from the last 12 years in roughly 120 tech stocks then saves it to an SQL database for use later. Link to video is here: 
In the future I will show you how to manipulate data, keep it up to date, and eventually run data processes on a cloud server


    # incase running on cloud server
    #!/usr/bin/python3

    #importing relevant libraries
    import pandas as pd
    import time
    import os
    import warnings
    import smtplib
    import datetime
    # pip install mysqlclient
    import MySQLdb as sql
    from time import sleep, strftime, localtime
    # pip install ibpy2 after downloading from online 
    from ib.opt import Connection, message
    from ib.ext.Contract import Contract
    from ib.ext.Order import Order
    from ib.ext.EWrapper import EWrapper
    from pandas import DataFrame
    from random import randint
    from datetime import datetime, timedelta
    # pip install sqlalchemy
    from sqlalchemy import create_engine

    # your email info
    EMAIL_ADDRESS = ''
    PASSWORD = ''

    # function for sending email
    class send:
        def send_mail(subject, msg):
            try:
                server = smtplib.SMTP('smtp.gmail.com:587')
                server.ehlo()
                server.starttls()
                server.login(EMAIL_ADDRESS, PASSWORD)
                message = 'Subject: {}{}'.format(subject, msg)
                server.sendmail(EMAIL_ADDRESS, EMAIL_ADDRESS, message)
                server.quit
                print("success sending email")
            except:
                print("email failed to send")

    #ignore warnings
    warnings.filterwarnings('ignore')

    #set up conneection to sql database
    engine = create_engine("mysql+mysqldb://root:@localhost/stocks") 

    stocks =['AAL','AAPL','ADBE','ADI','ADP','ADSK','AKAM','ALGN','ALXN','AMAT','AMGN','AMZN','ATVI','AVGO','BIDU','BIIB','BMRN','CA','CELG','CERN','CHKP','CHTR','CTRP','CTAS','CSCO','CTXS','CMCSA','COST','CSX','CTSH','DISCA','DISCK','DISH','DLTR','EA','EBAY','ESRX','EXPE','FAST','FB','FISV','FOX','FOXA','GILD','GOOG','GOOGL','HAS','HSIC','HOLX','ILMN','INCY','INTC','INTU','ISRG','JBHT','JD','KLAC','KHC','LBTYK','LILA','LBTYA','QRTEA','MELI','MAR','MAT','MDLZ','MNST','MSFT','MU','MXIM','MYL','NCLH','NFLX','NTES','NVDA','PAYX','BKNG','PYPL','QCOM','REGN','ROST','SHPG','SIRI','SWKS','SBUX','SYMC','TSCO','TXN','TMUS','ULTA','VIAB','VOD','VRTX','WBA','WDC','XRAY','IDXX','LILAK','LRCX','MCHP','ORLY','PCAR','STX','TSLA','VRSK','WYNN','XLNX']

    time_received_next_valid_order_id = None 
    next_valid_order_id = None
    number = randint(0, 10000000)

    for y in range(len(stocks)):
        try:
            c = stocks[y]
            print(c)

            def next_valid_order_id_handler(msg):
                global time_received_next_valid_order_id, next_valid_order_id
                next_valid_order_id = msg.orderId
                time_received_next_valid_order_id = time.time()

            def get_next_valid_order_id(con):
                global time_received_next_valid_order_id, next_valid_order_id
                last_time = time_received_next_valid_order_id
                next_valid_order_id = con.reqIds(1) # Always keep arg set to 1 (cruft)
                while last_time == time_received_next_valid_order_id:
                    time.sleep(2)
                return(next_valid_order_id)   

            low1 = []
            high1 = []
            open1 = [] 
            close1 = []
            volume1 = []
            date1 = []

            def data(msg): 
                close1.insert(len(close1), msg.close)
                volume1.insert(len(volume1), msg.volume)
                date1.insert(len(date1), msg.date)
                low1.insert(len(low1), msg.low)
                high1.insert(len(high1), msg.high)
                open1.insert(len(open1), msg.open)

            def make_contract(symbol): 
                contract = Contract()
                Contract.m_localSymbol = symbol
                Contract.m_exchange = 'SMART'
                Contract.m_secType = 'STK'
                Contract.m_primaryExch = 'SMART'
                Contract.m_currency = 'USD'
                return(contract)

            def make_order(action, quantity):
                order = Order()
                order.m_orderType = 'MKT'
                order.m_action = action
                order.m_transmit = True
                order.m_totalQuantity = quantity
                return(order)             

            def func(): 
                endtime = strftime('%Y%m%d %H:%M:%S') 
                con = Connection.create('localhost', port=1429, clientId=number)
                con.connect()
                con.register(data, message.historicalData)
                con.register(next_valid_order_id_handler, 'NextValidId')
                oid = get_next_valid_order_id(con)
                start = datetime.strptime('2006-01-01 01:01:01', "%Y-%m-%d %H:%M:%S")
                stop = datetime.strptime('2018-10-21 01:01:01' , "%Y-%m-%d %H:%M:%S")
                time.sleep(1)

                while start < stop:
                    start = start + timedelta(days=365)  # increase day one by one
                    qq = datetime.strftime(start, '%Y%m%d %H:%M:%S')
                    print(qq)

                    try: 
                        low1.clear()
                        high1.clear()
                        close1.clear()
                        open1.clear()
                        volume1.clear()
                        date1.clear()
                        con.reqHistoricalData(1, make_contract(str(c)), str(qq), "1 Y","1 day","TRADES",1,1)
                        time.sleep(5) 

                        for q in range(len(close1)-1): 

                            if q == 0:
                                data3 = {'closes': [close1[q]], 'lows': [low1[q]], 'highs': [high1[q]], 'opens': [open1[q]],
                                         'volumes': [volume1[q]],'dates': [date1[q]]}
                                df = pd.DataFrame(data3, columns=['closes','lows','highs','opens','volumes','dates'])

                            else:
                                df.loc[len(df)] = [close1[q],low1[q],high1[q],open1[q],volume1[q],date1[q]] 

                        if len(close1) == 0:
                            print('no data2')

                        else:
                            print(df)

                        df.to_sql(name=str(c), con=engine, if_exists='append', index=False)
                        df = df.iloc[0:0]
                        low1.clear()
                        high1.clear()
                        close1.clear()
                        open1.clear()
                        volume1.clear()
                        date1.clear()

                    except:
                        print('no data1')

                con.unregister(data, message.historicalData)
                con.disconnect()

                if str(c) == 'XRAY':
                    send.send_mail('stocks data collection', 'done')

            func()

        except:
            print('fail')







    
