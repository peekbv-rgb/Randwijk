"""VD-model (g/m3): voorspel de hoogste vochtdeficit van de dag (10-18u) uit ochtendcondities."""
import pandas as pd, numpy as np, json
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import mean_absolute_error, r2_score, mean_squared_error

d = pd.read_parquet("../data/weer_clean.parquet").set_index("dt").sort_index()
def es_kpa(T): return 0.6108*np.exp(17.27*T/(T+237.3))
def vd_gm3(T,RH):                      # vochtdeficit in g/m3
    es=es_kpa(T)*1000.0; ea=es*np.clip(RH,0,100)/100.0; Tk=T+273.15
    return 2.16679*(es-ea)/Tk
def dp(T,RH):
    a,b=17.62,243.12; g=np.log(np.clip(RH,1,100)/100)+a*T/(b+T); return b*g/(a-g)

d["vd"]=vd_gm3(d.temp,d.hum)
h=d[["temp","hum","wind","vd"]].resample("1h").mean(); h["date"]=h.index.normalize(); h["hour"]=h.index.hour
rows=[]
for day,g in h.groupby("date"):
    morn=g[g.hour==8]; daytime=g[(g.hour>=10)&(g.hour<=18)]
    if morn.empty or daytime["vd"].notna().sum()<5: continue
    m=morn.iloc[0]
    if pd.isna(m.temp) or pd.isna(m.hum): continue
    vmax=daytime["vd"].max(); tmax=g[(g.hour>=9)&(g.hour<=19)]["temp"].max()
    if pd.isna(vmax) or pd.isna(tmax): continue
    td=dp(m.temp,m.hum)
    rows.append({"date":day,"t_morn":m.temp,"rh_morn":m.hum,"td_morn":td,"wind_morn":m.wind,
       "vd_morn":m.vd,"dewdep_morn":m.temp-td,"tmax_day":tmax,"doy":day.dayofyear,"vd_max":vmax})
vd=pd.DataFrame(rows).dropna().reset_index(drop=True)
vd["doy_sin"]=np.sin(2*np.pi*vd.doy/365);vd["doy_cos"]=np.cos(2*np.pi*vd.doy/365)
vd["vdmax_prev"]=vd["vd_max"].shift(1); vd=vd.dropna().reset_index(drop=True)
print("Dagen:",len(vd),"| VD_max %.1f-%.1f g/m3 gem %.1f"%(vd.vd_max.min(),vd.vd_max.max(),vd.vd_max.mean()))
tr=vd[vd.date.dt.year<=2024]; te=vd[vd.date.dt.year==2025]
def fit(FEATS,name,out):
    sc=StandardScaler().fit(tr[FEATS]); lin=LinearRegression().fit(sc.transform(tr[FEATS]),tr.vd_max)
    p=lin.predict(sc.transform(te[FEATS]))
    print("%-9s MAE=%.2f g/m3  R2=%.3f"%(name,mean_absolute_error(te.vd_max,p),r2_score(te.vd_max,p)))
    json.dump({"feats":FEATS,"mean":sc.mean_.tolist(),"scale":sc.scale_.tolist(),
               "lin_coef":lin.coef_.tolist(),"lin_int":float(lin.intercept_)}, open(out,"w"), indent=1)
fit(["t_morn","rh_morn","td_morn","wind_morn","vd_morn","tmax_day","vdmax_prev","doy_sin","doy_cos"],"fc(tmax)","../models/vd_model_tmax.json")
fit(["t_morn","rh_morn","td_morn","wind_morn","vd_morn","dewdep_morn","vdmax_prev","doy_sin","doy_cos"],"ochtend","../models/vd_model_morning.json")
vd.to_csv("../data/Randwijk_dagen_VD_kenmerken.csv",index=False,float_format="%.2f")
print("Klaar.")
