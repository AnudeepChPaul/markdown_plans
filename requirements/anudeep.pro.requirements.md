I want to create 3 apps

I want to crate a central identity management which will create tokens and other services will call identiy to validate them. identity will also manage user system. It's the central identity service.



1. iam.anudeep.pro: /Users/anudeepchandrapaul/Projects/github.com/AnudeepChPaul/iam.anudeep.pro;
    here this is an identity management service. Should be written in python and should use FastAPI as it as fastest runtime.
    1. It will have following endpoints.
        /v1/register
            register will receive email/password and option to opt in for authenticator code (2way verification)
        /v1/register/totp
            if user opts in 2step verification, then totp will present a qr code to setup authenticator code. otherwise, this step never comes. for each user, there will be separate secret to generate the totp; the secret should be different for each user, so that totp generation stays different for different users. 

        Registration will happen based on a flag called IAM_REGISTRATION_MODE='open', 'closed', 'single'

        if mode is open; then any number of users will be able to register
        if mode is closed; then no one will be able to register
        if mode is single; only 1 single user will be able to register.

        /v1/login
            Once user registers, they'll be able to login. login will take email/password from req.body, and X-Pro- prefixed headers to identify the particular user. Like email, user-ip, user-agent, and other headers to particularly identify an user. Login will generate an temporary token & a fingerprint (stored on redis) and against the token it'll see if 2 way auth is enabled.
        /v1/login/totp
            Then if 2 way auth is enabled, then user will provide just the totp here along with X-Pro headers (to match the fingerprint) and validate totp and fingerprint. If it matches fine, then user will get a 256hash key which for other apps, it'll keep on passing to validate the authenticity of the API call.

            Once logged in, 256hash key will the point of source for user's identification and fingerprint will be point of source for user's authenticity.
        /v1/logout


2. admin.anudeep.pro -> There will be another single admin portal for different system. Right now, we will have a tab on admin portal called resume and that'll be our third app. Resume on admin portal will have different options called Title, experiences, projects, technologies, and bio. All of them will be driven by markdown files and admin portal will have an markdown editor to update them live. Once saved, admin will process them for static page generation. tech stack for this will be on nodejs and react. It doesn't need to be ssr based app. But, this also needs to be blazing fast with keyboard shortcut support for faster navigation. the backend for this must support graphql and for file upload it can work with REST API(s). 


3. resume.anudeep.pro--> Now, resume app will take these data and display it on the resume site. Resume site shoudl be blazing fast but will also be dynamic based on admin>resume updates. the background of the resume site will be a 3d animated model and will move based on the cursor movements. The 3d model can load a bit later but the resume details should load blazing fast and user should get fastest painting of the side to 3-4ms and time to interactive is less than 5ms. background 3d model can load in 100-200ms and once loaded will be moving based on cursor movements. Should have beautiful animations with lovely transitions. Should have a few sections as follows My Professional experiences, technologies, projects.[27;5;106~
