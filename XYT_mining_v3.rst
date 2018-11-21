
XYT mining op de weg
====================

In deze project identificeren we start- en stoplocaties op basis van XYT
gegevens.

.. code:: ipython3

    ##import packages
    import pandas as pd
    pd.set_option('mode.chained_assignment', None)
    import pandas as pd
    import os
    import fnmatch
    import concurrent.futures
    import numpy as np
    from geopy.distance import geodesic
    #from geopy.distance import great_circle
    #from geopy.distance import vincenty

Eerst voegen we twee dagen aan XYT data samen

.. code:: ipython3

    ## Import files
    path = 'type-hier-je-pad-naar-bestanden'
    
    with concurrent.futures.ProcessPoolExecutor() as executor:
        configfiles = [os.path.join(dirpath, f)
            for dirpath, dirnames, files in os.walk(path)
            for f in fnmatch.filter(files, '*.csv')]
            
    list = []
    for file in configfiles:
        df = pd.read_csv(file,index_col=None, header=0, usecols=[0,1,2,3])
        list.append(df)
        df = pd.concat(list)
        
    ##drop index + create index
    df.reset_index(inplace=True)
    df['index'] = df.index

Daarna bereken we de afstand tussen twee XY punten aan de hand van de
geodesics formule van Karney (2013). Deze formule zit in de package
Geopy.

.. code:: ipython3

    ##New table: van a naar b --> drop na 
    df = df.rename(index=str, columns={"Lon": "Lon_a", "Lat": "Lat_a"})
    df["Lat_b"] = df["Lat_a"].shift(-1)
    df["Lon_b"] = df["Lon_a"].shift(-1)
    df = df.dropna()
    
    ##Measure distance
    def distancer_km(row):
        coords_1 = (row['Lat_a'], row['Lon_a'])
        coords_2 = (row['Lat_b'], row['Lon_b'])
        return geodesic(coords_1, coords_2).km
        #return vincenty(coords_1, coords_2).km
    
    def distancer_m(row):
        coords_1 = (row['Lat_a'], row['Lon_a'])
        coords_2 = (row['Lat_b'], row['Lon_b'])
        return geodesic(coords_1, coords_2).m
        #return vincenty(coords_1, coords_2).km
    
    df['distance_km'] = df.apply(distancer_km, axis=1)
    df['distance_m'] = df.apply(distancer_m, axis=1)

Nu gaan we het verschil in seconden tussen twee XY punten berekenen.
Hiervoor gebruiken we de package Numpy.

.. code:: ipython3

    ##Change date format
    df["date_a"] = np.array(df["Dt"], dtype="datetime64")
    
    ##difference in seconds between two XY
    df["date_b"] = df["date_a"]
    df["date_b"] = df["date_b"].shift(-1)
    df["date_b"] = df["date_b"].dropna()
    df["diff"] = df["date_b"] - df["date_a"]
    df["diff_sec"] = df["diff"].astype('timedelta64[s]')


.. parsed-literal::

    C:\Users\candy\Anaconda3\lib\site-packages\ipykernel_launcher.py:2: DeprecationWarning: parsing timezone aware datetimes is deprecated; this will raise an error in the future
      
    

Door het verschil in seconden/uur te delen door verschil in
meters/kilometers kunnen we de snelheid berekenen.

.. code:: ipython3

    ## meters per second / km per hour
    df["speed_ms"] = df["distance_m"]/df["diff_sec"]
    df["speed_kmu"] = df["distance_km"]/df["diff_sec"].divide(60*60)


