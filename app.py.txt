import streamlit as st
import requests
import folium
from geopy.geocoders import Nominatim
from datetime import datetime
from streamlit_folium import st_folium

st.title("GOBRUH - Ride Booking AI")

geolocator = Nominatim(user_agent="grab_ai_pro")

start_place = st.text_input("Điểm đi")
end_place = st.text_input("Điểm đến")

vehicle = st.selectbox("Chọn xe", ["bike","car"])
weather_type = st.selectbox("Thời tiết", ["đẹp","bình thường","rất xấu"])

promo_code = st.text_input("Mã khuyến mãi (nếu có)")

discount = 0
valid_codes = ["GIAMGIA15","DEAL15"]

if promo_code.upper() in valid_codes:
    discount = 0.15

def get_surge_factor():
    hour = datetime.now().hour
    if 6 <= hour < 9:
        return 1.4,0.6
    elif 16 <= hour < 19:
        return 1.6,0.5
    else:
        return 1.0,1.0

def weather_factor(w):
    if w=="rất xấu":
        return 1.3
    elif w=="bình thường":
        return 1.15
    else:
        return 1.0

if st.button("Tính giá chuyến đi"):

    start_loc = geolocator.geocode(start_place)
    end_loc = geolocator.geocode(end_place)

    start_point=(start_loc.latitude,start_loc.longitude)
    end_point=(end_loc.latitude,end_loc.longitude)

    start_lon_lat=f"{start_point[1]},{start_point[0]}"
    end_lon_lat=f"{end_point[1]},{end_point[0]}"

    url=f"http://router.project-osrm.org/route/v1/driving/{start_lon_lat};{end_lon_lat}?overview=full&geometries=geojson"

    r=requests.get(url).json()
    route=r["routes"][0]

    distance_km=route["distance"]/1000
    time_min=route["duration"]/60

    coords=route["geometry"]["coordinates"]
    route_coords=[(c[1],c[0]) for c in coords]

    surge,speed=get_surge_factor()
    weather=weather_factor(weather_type)

    if vehicle=="bike":
        base=12000
        per_km=4200
        per_min=344
        vehicle_factor=1
    else:
        base=28000
        per_km=9800
        per_min=442
        vehicle_factor=1.6

    actual_time=time_min*(1/speed)

    price=(base+distance_km*per_km+actual_time*per_min)
    price=price*surge*weather*vehicle_factor
    price=price*(1-discount)

    st.write("Khoảng cách:",round(distance_km,2),"km")
    st.write("Thời gian:",round(actual_time,1),"phút")
    st.write("Giá chuyến đi:",format(round(price),","),"VND")

    mid=((start_point[0]+end_point[0])/2,(start_point[1]+end_point[1])/2)

    m=folium.Map(location=mid,zoom_start=13)

    folium.PolyLine(route_coords,color="blue").add_to(m)
    folium.Marker(start_point,popup="Điểm đi").add_to(m)
    folium.Marker(end_point,popup="Điểm đến").add_to(m)

    st_folium(m,width=700)