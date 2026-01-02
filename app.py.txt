import streamlit as st import pandas as pd import re import folium from streamlit_folium import st_folium from math import radians, cos, sin, asin, sqrt from PIL import Image

--- تحميل الشعار وQR Code ---

logo = Image.open("masartawsil_logo.png") qr_code = Image.open("masartawsil_qrcode.png")

st.set_page_config(page_title="مسار التوصيل / Masar Al Tawseel", layout="wide") st.image([logo, qr_code], width=150) st.title("مسار التوصيل / Masar Al Tawseel")

--- إدخال نص الواتساب ---

user_input = st.text_area("الصق رسائل الواتساب هنا / Paste WhatsApp messages here", height=200)

--- Regex لاستخراج الأرقام والروابط ---

phone_pattern = r"(05\d{8})" map_pattern = r"https?://[\w./?=&-]+" coord_pattern = r"(-?\d{1,3}.\d+)[,\s]+(-?\d{1,3}.\d+)"

--- قائمة لتخزين العملاء ---

data = [] if st.button("📌 استخراج البيانات / Extract Data"): phones = re.findall(phone_pattern, user_input) links = re.findall(map_pattern, user_input) coords_text = re.findall(coord_pattern, user_input)

for p, l in zip(phones, links):
    data.append({"رقم الجوال / Phone": p, "رابط الخريطة / Map Link": l, "lat": "", "lng": ""})
for coord in coords_text:
    lat, lng = map(float, coord)
    data.append({"رقم الجوال / Phone": "", "رابط الخريطة / Map Link": "", "lat": lat, "lng": lng})

if data:
    st.session_state.df = pd.DataFrame(data)
    st.success("✔ تم استخراج العملاء بنجاح / Data extracted successfully!")
else:
    st.warning("⚠ لم يتم العثور على بيانات / No data found")

--- إدخال نقطة البداية ---

start_location = st.text_input("📌 نقطة البداية / Start Location (Lat,Lng or Google Maps link)")

--- دالة لحساب المسافة ---

def haversine(lat1, lon1, lat2, lon2): lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2]) dlat = lat2 - lat1 dlon = lon2 - lon1 a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2 c = 2 * asin(sqrt(a)) km = 6371 * c return km

--- ترتيب حسب المسافة ---

if 'df' in st.session_state and start_location and st.button("🗺️ ترتيب حسب الأقرب / Optimize Route"): df = st.session_state.df.copy() try: if ',' in start_location: start_lat, start_lng = map(float, start_location.split(',')) else: st.warning("❌ أدخل الإحداثيات بصيغة Lat,Lng / Enter as Lat,Lng") st.stop() distances = [] for idx, row in df.iterrows(): if row['lat'] != '' and row['lng'] != '': distances.append(haversine(start_lat, start_lng, float(row['lat']), float(row['lng']))) else: distances.append(9999) df['distance'] = distances df = df.sort_values(by='distance').reset_index(drop=True) st.session_state.df = df.drop(columns=['distance']) st.success("✔ تم ترتيب المسار حسب الأقرب / Route optimized!") except: st.warning("❌ خطأ في الإحداثيات / Invalid coordinates")

--- عرض الجدول مع إمكانية التعديل ---

if 'df' in st.session_state: df = st.session_state.df.copy() edited_df = st.experimental_data_editor(df, num_rows="dynamic")

# --- أزرار Google Maps و WhatsApp ---
for i, row in edited_df.iterrows():
    col1, col2 = st.columns(2)
    if row['رابط الخريطة / Map Link']:
        col1.button(f"🌍 فتح في قوقل ماب {i+1} / Open Map {i+1}", key=f'map{i}', on_click=lambda url=row['رابط الخريطة / Map Link']: st.experimental_set_query_params(url=url))
    if row['رقم الجوال / Phone']:
        wa_link = f"https://wa.me/{row['رقم الجوال / Phone']}"
        col2.button(f"💬 واتساب {i+1} / WhatsApp {i+1}", key=f'wa{i}', on_click=lambda url=wa_link: st.experimental_set_query_params(url=url))

--- خريطة تفاعلية مع الترقيم الدائري ---

if 'df' in st.session_state and not st.session_state.df.empty: m = folium.Map(location=[24.7136, 46.6753], zoom_start=6) for idx, r in st.session_state.df.iterrows(): if r['lat'] != '' and r['lng'] != '': folium.CircleMarker( location=[float(r['lat']), float(r['lng'])], radius=12, color='blue', fill=True, fill_color='blue', fill_opacity=0.7, tooltip=f"{idx+1}. {r['رقم الجوال / Phone']}" ).add_to(m) st_folium(m, width=700, height=400)

--- اختيار الثيم ---

theme_choice = st.selectbox("اختر الثيم / Choose Theme", ["أخضر / Green", "أزرق / Blue", "داكن / Dark"]) st.write(f"ثيم مختار / Selected theme: {theme_choice}")

st.info("✔ كل المزايا مفعّلة: الجدول، الخريطة، ترتيب المسار، الأزرار، الترقيم الدائري، نقطة البداية قابلة للتحديد، واجهة ثنائية اللغة")