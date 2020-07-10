# -*- coding: utf-8 -*-
"""
Created on Sat Jun 13 10:45:10 2020

@author: FALAH FAKHRI

Module: Read_And_Map_Tropomi_NO2.py
========================================================================================================
This is an educational and applicalable code to extract, retrieve and map the NO2 tropospheric emitted
gas, of multiple single product (offline or/and real time products). 
Please see the README associated with this module for more information 
========================================================================================================

"""

try:
    
    from netCDF4 import Dataset
    import numpy as np
    import matplotlib.pyplot as plt
    from matplotlib.colors import LogNorm
    from mpl_toolkits.basemap import Basemap
    from glob import iglob
    from os.path import join
    from termcolor import colored
    import sys
    import os
    import datetime
    from datetime import date
    import re
        
except ModuleNotFoundError:
    print('Module improt error')
    sys.exit()
else:
    print(colored('\nAll libraries properly loaded. Ready to start!!!', 'green'), '\n')
    
# Read the data from any directory
   
user_input_dir = input('\nPlease enter your directory path\n\nDirectory path:\n')
product_path = os.path.join('product_path', user_input_dir)  

# lOOK FOR NO2 products in target directory

try:
    
    input_files_OFFL = sorted(list(iglob(join(product_path, '**', '*OFFL*NO2*.nc'), recursive=True)))
    input_files_RPRO = sorted(list(iglob(join(product_path, '**', '*RPRO*NO2*.nc'), recursive=True)))
    print(colored('NO2 OFFL products detected:', 'blue'), len(input_files_OFFL))
    print(colored('NO2 NRTI products detected:', 'blue'), len(input_files_RPRO))
    
    if len(input_files_OFFL) == 0:
        s5p_file = input_files_RPRO [0]
    else:
        s5p_file = input_files_OFFL[0]
    
except:
    print('Did not find a product')
    
else:
    
    print(colored('\nProduct selected for anlayses:\n\n', 'blue'), s5p_file)

# Read the product
for FILE in input_files_OFFL:
    FILE_NAME = FILE.strip()
    user_input = input('\nWould you like to process\n' + FILE_NAME + '\n\n(Y/N)')
    if(user_input == 'N' or user_input == 'n'):
        print('Skipping...')
        continue
    elif(user_input == 'Y' or user_input == 'y'):
        print('Processing...')
        
        # Read the *.nc file
        read_product = Dataset(FILE, mode='r')
        print(read_product)
        
        # Slice sentinel 5p to a 2D array using use the [0,:,:] code
        lons = read_product.groups['PRODUCT'].variables['longitude'][:][0,:,:]
        lats = read_product.groups['PRODUCT'].variables['latitude'][:][0,:,:]
        no2 = read_product.groups['PRODUCT'].variables['nitrogendioxide_tropospheric_column_precision'][0,:,:]
        print (lons.shape)
        print (lats.shape)
        print (no2.shape)
        
        # The multiplication factor to convert mol/m2to molec/cm2 is 6.02214×1019.
        no2_units = read_product.groups['PRODUCT'].variables['nitrogendioxide_tropospheric_column_precision'].units * 6,136.56066
        
        # Use base map library to map the NO2
        lon_0 = lons.mean()
        lat_0 = lats.mean()
        fig = plt.figure(figsize=(12, 12))
        mp = Basemap(width=1195000,height=950000,
            resolution='l',projection='stere',\
            lat_0=33, lon_0=43)
    xi, yi = mp(lons, lats)
    
    # Plot Data
    cs = mp.pcolor(xi,yi,np.squeeze(no2),norm=LogNorm(), cmap='jet')
    
    # Add Grid Lines
    mp.drawparallels(np.arange(-180., 80., 43.), labels=[1,0,0,0], fontsize=10)
    mp.drawmeridians(np.arange(-180., 80., 33.), labels=[0,0,0,1], fontsize=10)
    my_cmap = plt.cm.get_cmap('gist_stern_r')
    my_cmap.set_under('w')
   
    # Add Coastlines, States, and Country Boundaries
    mp.drawcoastlines()
    mp.drawstates()
    mp.drawcountries(linewidth = 1, linestyle='solid', color = 'k')
    mp.drawrivers()
    
    #add city of Baghdad
    x, y = mp(44.3661, 33.3152)
    plt.plot(x, y, 'or', markersize=8)
    plt.text(x, y, ' Baghdad', fontsize=10,  color='red')
    
    # add city of Mosul
    x1, y1 = mp(43.130001, 36.340000)
    plt.plot(x1, y1, 'or', markersize=8)
    plt.text(x1, y1, ' Mosul', fontsize=10,  color='red')
   
    # Add Colorbar
    cbar = mp.colorbar(cs, location='bottom', pad="10%", label="molec cm^-2")
    
    # Add Title (The title is represented by the acquisition date)
    imgti = FILE
    match = re.search(r'\d{8}', imgti)
    date = datetime.datetime.strptime(match.group(), '%Y%m%d').date()
    plt.title(date)
    plt.show()
    
    #once you close the map it asks if you'd like to save it
    is_save=str(input('\nWould you like to save this map? Please enter Y or N \n'))
    if is_save == 'Y' or is_save == 'y':
        #saves as a png if the user would like
        pngfile = '{0}.png'.format(FILE_NAME[:-3])
        fig.savefig(pngfile, dpi = 300)
    
    #close the netCDF4 file 
read_product.close()
