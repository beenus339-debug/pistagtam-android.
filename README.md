# pistagtam-android.
Purple and White Social Media App 

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>PISTAGTAM</title>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<style>
*{box-sizing:border-box}
body{margin:0;background:#f7f4ff;color:#29213b;font-family:Arial,sans-serif}
header{background:#7138c9;color:white;padding:17px;text-align:center;font-size:25px;font-weight:bold;letter-spacing:1px}
main{max-width:600px;margin:auto;padding:15px}
.panel,.post{background:white;border-radius:16px;padding:16px;margin:14px 0;box-shadow:0 3px 14px #29114b12}
h2{color:#7138c9;margin-top:4px}
input,textarea,button{width:100%;padding:12px;margin:7px 0;border-radius:10px;font-size:15px}
input,textarea{border:1px solid #d8c9ed;background:white}
button{border:0;background:#7138c9;color:white;font-weight:bold;cursor:pointer}
button.secondary{background:#eee7fa;color:#59309a}
button:disabled{opacity:.5}
.hidden{display:none}
.muted{color:#776d85;font-size:13px}
.post img{width:100%;max-height:500px;object-fit:cover;border-radius:12px}
.posthead{font-weight:bold;margin-bottom:10px}
.actions{display:flex;gap:8px}
.actions button{width:auto;flex:1}
.comment{padding:7px 0;border-top:1px solid #eee}
#message{padding:10px;color:#6332a8}
</style>
</head>
<body>
<header>PISTAGTAM</header>
<main>
<div id="message">Welcome to PISTAGTAM</div>

<section class="panel" id="auth">
<h2>Welcome</h2>
<input id="email" type="email" placeholder="Email address">
<input id="password" type="password" placeholder="Password">
<input id="username" placeholder="Username (for new account)">
<button id="signup">Create account</button>
<button class="secondary" id="login">Login</button>
<p class="muted">Use an email address you control. Email confirmation may be required.</p>
</section>

<section class="panel hidden" id="app">
<div>
<strong id="myname"></strong>
<button class="secondary" id="logout">Log out</button>
</div>
<h2>Share a photo</h2>
<input id="photo" type="file" accept="image/*">
<textarea id="caption" maxlength="500" placeholder="Write a caption..."></textarea>
<button id="upload">Upload photo</button>
<h2>Home feed</h2>
<button class="secondary" id="refresh">Refresh feed</button>
<div id="feed"></div>
</section>
</main>

<script>
const SUPABASE_URL = "https://kkgbsvtyiygiiuikjfwn.supabase.co";
const SUPABASE_KEY = "sb_publishable_sYX7FyMBTF0FjyOwr8f0Kg_v3rVQKcf";
const db = window.supabase.createClient(SUPABASE_URL,SUPABASE_KEY);

let currentUser=null;
const $=id=>document.getElementById(id);
function message(t){$("message").textContent=t}
function safe(t){return String(t??"").replace(/[&<>"']/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c]))}

async function ensureProfile(user){
 const {data,error}=await db.from("profiles").select("*").eq("id",user.id).maybeSingle();
 if(error) throw error;
 if(data) return data;
 const suggested=(user.email||"member").split("@")[0].replace(/[^a-zA-Z0-9_]/g,"").slice(0,18)||"member";
 const username=suggested+"_"+user.id.slice(0,5);
 const {data:created,error:insertError}=await db.from("profiles")
  .insert({id:user.id,username,full_name:suggested}).select().single();
 if(insertError) throw insertError;
 return created;
}

async function showApp(){
 const {data:{session}}=await db.auth.getSession();
 currentUser=session?.user||null;
 $("auth").classList.toggle("hidden",!!currentUser);
 $("app").classList.toggle("hidden",!currentUser);
 if(currentUser){
  try{
   const p=await ensureProfile(currentUser);
   $("myname").textContent="@"+p.username;
   await loadFeed();
  }catch(e){message("Profile error: "+e.message)}
 }
}
$("signup").onclick=async()=>{
 try{
  const email=$("email").value.trim(),password=$("password").value,username=$("username").value.trim();
  if(!email||password.length<6||!username){message("Enter email, username and password (at least 6 characters).");return}
  const {data,error}=await db.auth.signUp({email,password});
  if(error) throw error;
  if(data.session){
   const {error:pe}=await db.from("profiles").insert({id:data.user.id,username,full_name:username});
   if(pe) throw pe;
   message("Account created!");
   await showApp();
  }else{
   message("Check your email to confirm your account, then log in. Your profile will be created after login if needed.");
  }
 }catch(e){message("Sign up error: "+e.message)}
};
$("login").onclick=async()=>{
 try{
  const {error}=await db.auth.signInWithPassword({email:$("email").value.trim(),password:$("password").value});
  if(error) throw error;
  message("Logged in!");
  await showApp();
 }catch(e){message("Login error: "+e.message)}
};
$("logout").onclick=async()=>{await db.auth.signOut();message("Logged out");await showApp()};
$("refresh").onclick=loadFeed;

$("upload").onclick=async()=>{
 try{
  if(!currentUser) return;
  const file=$("photo").files[0];
  if(!file){message("Choose a photo first.");return}
  if(!file.type.startsWith("image/")){message("Please select an image.");return}
  if(file.size>5*1024*1024){message("Choose an image smaller than 5 MB.");return}
  $("upload").disabled=true;
  message("Uploading photo...");
  const path=currentUser.id+"/"+Date.now()+"_"+file.name.replace(/[^a-zA-Z0-9._-]/g,"_");
  const {error:upError}=await db.storage.from("post-images").upload(path,file,{upsert:false});
  if(upError) throw upError;
  const {data:urlData}=db.storage.from("post-images").getPublicUrl(path);
  const {error:postError}=await db.from("posts").insert({
   user_id:currentUser.id,image_url:urlData.publicUrl,caption:$("caption").value.trim()
  });
  if(postError) throw postError;
  $("photo").value="";$("caption").value="";
  message("Photo uploaded!");
  await loadFeed();
 }catch(e){message("Upload error: "+e.message)}
 finally{$("upload").disabled=false}
};

async function loadFeed(){
 try{
  $("feed").textContent="Loading posts...";
  const {data:posts,error}=await db.from("posts").select("*").order("created_at",{ascending:false}).limit(50);
  if(error) throw error;
  if(!posts.length){$("feed").textContent="No posts yet. Share the first photo!";return}
  const ids=[...new Set(posts.map(p=>p.user_id))];
  const {data:profiles,error:profileError}=await db.from("profiles").select("id,username").in("id",ids);
  if(profileError) throw profileError;
  const names={};(profiles||[]).forEach(p=>names[p.id]=p.username);
  const postIds=posts.map(p=>p.id);
  const {data:likes,error:likesError}=await db.from("likes").select("post_id,user_id").in("post_id",postIds);
  if(likesError) throw likesError;
  const {data:comments,error:commentsError}=await db.from("comments").select("id,post_id,user_id,comment").in("post_id",postIds).order("created_at");
  if(commentsError) throw commentsError;
  $("feed").innerHTML=posts.map(p=>{
   const pl=(likes||[]).filter(l=>l.post_id===p.id);
   const pc=(comments||[]).filter(c=>c.post_id===p.id);
   const liked=pl.some(l=>l.user_id===currentUser.id);
   return `<article class="post">
    <div class="posthead">@${safe(names[p.user_id]||"member")}</div>
    <img src="${safe(p.image_url)}" alt="Shared photo" loading="lazy">
    <p>${safe(p.caption)}</p>
    <div class="actions">
     <button class="secondary like" data-id="${safe(p.id)}">${liked?"♥ Liked":"♡ Like"} (${pl.length})</button>
    </div>
    <div><strong>Comments</strong></div>
    ${pc.map(c=>`<div class="comment">${safe(c.comment)}</div>`).join("")}
    <input class="commentInput" data-id="${safe(p.id)}" placeholder="Write a comment">
    <button class="secondary commentBtn" data-id="${safe(p.id)}">Send comment</button>
   </article>`;
  }).join("");
  document.querySelectorAll(".like").forEach(b=>b.onclick=()=>toggleLike(b.dataset.id));
  document.querySelectorAll(".commentBtn").forEach(b=>b.onclick=()=>addComment(b.dataset.id));
 }catch(e){$("feed").textContent="Could not load feed: "+e.message}
}

async function toggleLike(postId){
 try{
  const {data:existing,error}=await db.from("likes").select("post_id").eq("post_id",postId).eq("user_id",currentUser.id).maybeSingle();
  if(error) throw error;
  const result=existing
   ? await db.from("likes").delete().eq("post_id",postId).eq("user_id",currentUser.id)
   : await db.from("likes").insert({post_id:postId,user_id:currentUser.id});
  if(result.error) throw result.error;
  await loadFeed();
 }catch(e){message("Like error: "+e.message)}
}

async function addComment(postId){
 try{
  const input=[...document.querySelectorAll(".commentInput")].find(x=>x.dataset.id===postId);
  const comment=input?.value.trim();
  if(!comment)return;
  const {error}=await db.from("comments").insert({post_id:postId,user_id:currentUser.id,comment});
  if(error) throw error;
  message("Comment added!");
  await loadFeed();
 }catch(e){message("Comment error: "+e.message)}
}

db.auth.onAuthStateChange(()=>setTimeout(showApp,0));
showApp();
</script>
</body>
</html>
