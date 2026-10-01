import React,{useEffect,useState} from "react";
import {View,Text,TextInput,Pressable,ScrollView,StyleSheet,Alert} from "react-native";
import {StatusBar} from "expo-status-bar";
import {register,login,logout,getProfile,observeAuth} from "./src/auth";
import {createOrder,watchCustomerOrders,watchProviderOrders,acceptOrder,updateOrderStatus} from "./src/orders";

const services=[["نقل أثاث","🚚",149],["فك وتركيب","🛠️",99],["تحميل وتنزيل","📦",79],["توصيل قطعة","🛵",59]];

export default function App(){
 const [user,setUser]=useState(null),[profile,setProfile]=useState(null),[mode,setMode]=useState("login");
 useEffect(()=>observeAuth(async u=>{setUser(u);setProfile(u?await getProfile(u.uid):null)}),[]);
 if(!user) return <Auth mode={mode} setMode={setMode}/>;
 return profile?.role==="provider"
   ? <Provider user={user} profile={profile}/>
   : <Customer user={user} profile={profile}/>;
}

function Auth({mode,setMode}){
 const [name,setName]=useState(""),[email,setEmail]=useState(""),[password,setPassword]=useState(""),[role,setRole]=useState("customer"),[busy,setBusy]=useState(false);
 const submit=async()=>{try{setBusy(true); if(mode==="register") await register(email,password,name,role); else await login(email,password)}catch(e){Alert.alert("تعذر الدخول",e.message)}finally{setBusy(false)}};
 return <View style={s.auth}><StatusBar style="light"/><Text style={s.logo}>نَقْلة</Text><Text style={s.tag}>منصة الخدمات عند بابك</Text>
   {mode==="register"&&<TextInput style={s.input} placeholder="الاسم" value={name} onChangeText={setName}/>}
   <TextInput style={s.input} placeholder="البريد الإلكتروني" autoCapitalize="none" value={email} onChangeText={setEmail}/>
   <TextInput style={s.input} placeholder="كلمة المرور" secureTextEntry value={password} onChangeText={setPassword}/>
   {mode==="register"&&<View style={s.row}><Pressable style={[s.role,role==="customer"&&s.selected]} onPress={()=>setRole("customer")}><Text>عميل</Text></Pressable><Pressable style={[s.role,role==="provider"&&s.selected]} onPress={()=>setRole("provider")}><Text>مقدم خدمة</Text></Pressable></View>}
   <Pressable style={s.btn} onPress={submit} disabled={busy}><Text style={s.btnText}>{busy?"جارٍ التنفيذ…":mode==="register"?"إنشاء الحساب":"دخول"}</Text></Pressable>
   <Pressable onPress={()=>setMode(mode==="login"?"register":"login")}><Text style={s.link}>{mode==="login"?"ليس لديك حساب؟ إنشاء حساب":"لديك حساب؟ تسجيل الدخول"}</Text></Pressable>
 </View>
}

function Customer({user}){
 const [service,setService]=useState(services[0]),[from,setFrom]=useState(""),[to,setTo]=useState(""),[orders,setOrders]=useState([]);
 useEffect(()=>watchCustomerOrders(user.uid,setOrders),[]);
 const order=async()=>{if(!from||!to)return Alert.alert("أكمل البيانات","أدخل موقع الاستلام والتسليم");
   await createOrder({customerId:user.uid,service:service[0],from,to,price:service[2]}); Alert.alert("تم","تم إرسال الطلب لمقدمي الخدمة");
 };
 return <Screen title="نَقْلة" onLogout={logout}><Text style={s.h}>اختر الخدمة</Text><ScrollView horizontal showsHorizontalScrollIndicator={false}>{services.map(x=><Pressable key={x[0]} style={[s.service,service[0]===x[0]&&s.active]} onPress={()=>setService(x)}><Text style={s.icon}>{x[1]}</Text><Text style={s.bold}>{x[0]}</Text><Text>{x[2]} ر.س+</Text></Pressable>)}</ScrollView>
   <Text style={s.h}>تفاصيل الطلب</Text><TextInput style={s.input} placeholder="من" value={from} onChangeText={setFrom}/><TextInput style={s.input} placeholder="إلى" value={to} onChangeText={setTo}/>
   <Pressable style={s.btn} onPress={order}><Text style={s.btnText}>اطلب الآن — {service[2]} ر.س+</Text></Pressable>
   <Text style={s.h}>طلباتي</Text>{orders.map(o=><View style={s.order} key={o.id}><Text style={s.bold}>{o.service}</Text><Text>{o.from} ← {o.to}</Text><Text style={s.status}>{statusAr[o.status]||o.status}</Text></View>)}
 </Screen>
}

const statusAr={searching:"جاري البحث عن مقدم خدمة",accepted:"تم قبول الطلب",on_the_way:"في الطريق",started:"بدأت الخدمة",completed:"مكتمل",cancelled:"ملغي"};

function Provider({user}){
 const [orders,setOrders]=useState([]);
 useEffect(()=>watchProviderOrders(setOrders),[]);
 return <Screen title="نَقْلة — مقدم الخدمة" onLogout={logout}><Text style={s.h}>الطلبات المتاحة</Text>{orders.length===0&&<Text style={s.muted}>لا توجد طلبات متاحة الآن.</Text>}
 {orders.map(o=><View style={s.order} key={o.id}><Text style={s.bold}>{o.service} • {o.price} ر.س+</Text><Text>{o.from} ← {o.to}</Text><Pressable style={s.smallBtn} onPress={()=>acceptOrder(o.id,user.uid)}><Text style={s.btnText}>قبول الطلب</Text></Pressable></View>)}</Screen>
}

function Screen({children,title,onLogout}){return <View style={s.page}><View style={s.head}><Text style={s.headText}>{title}</Text><Pressable onPress={onLogout}><Text style={{color:"#fff"}}>خروج</Text></Pressable></View><ScrollView contentContainerStyle={{padding:18}}>{children}</ScrollView></View>}

const s=StyleSheet.create({
 page:{flex:1,backgroundColor:"#f5f6f8"},auth:{flex:1,justifyContent:"center",padding:22,backgroundColor:"#111827"},logo:{fontSize:42,fontWeight:"900",color:"#fff",textAlign:"center"},tag:{color:"#cbd5e1",textAlign:"center",marginBottom:30},input:{backgroundColor:"#fff,borderRadius:14,padding:15,marginBottom:10,borderWidth:1,borderColor:"#e5e7eb",fontSize:15},btn:{backgroundColor:"#111827",padding:16,borderRadius:15,alignItems:"center",marginVertical:8},btnText:{color:"#fff",fontWeight:"800"},link:{color:"#fff",textAlign:"center",marginTop:12},row:{flexDirection:"row",gap:8,marginBottom:8},role:{flex:1,padding:14,backgroundColor:"#fff",borderRadius:12,alignItems:"center"},selected:{borderWidth:2,borderColor:"#60a5fa"},head:{backgroundColor:"#111827",paddingTop:52,paddingBottom:18,paddingHorizontal:18,flexDirection:"row",justifyContent:"space-between"},headText:{fontSize:23,fontWeight:"900",color:"#fff"},h:{fontSize:19,fontWeight:"900",marginTop:14,marginBottom:12},service:{backgroundColor:"#fff",padding:14,borderRadius:16,width:145,marginRight:10,borderWidth:1,borderColor:"#e5e7eb"},active:{backgroundColor:"#e5e7eb",borderColor:"#111827"},icon:{fontSize:28},bold:{fontWeight:"800",marginBottom:5},order:{backgroundColor:"#fff",padding:16,borderRadius:16,marginBottom:10,borderWidth:1,borderColor:"#e5e7eb"},status:{marginTop:8,fontWeight:"700"},muted:{color:"#6b7280"},smallBtn:{backgroundColor:"#111827",padding:12,borderRadius:12,alignItems:"center",marginTop:10}
});
