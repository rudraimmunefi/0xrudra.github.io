Reviewing the code for app.py, I discovered that they are using a weird captcha where we are given first 5 bytes of a md5 and we needed to provide a valid string for it.

I used the following script to get around it.

```python3
import hashlib
import random
import string
import sys

def random_string(length):
    return ''.join(random.choices(string.ascii_lowercase, k=length))

def main():
    if len(sys.argv) != 3:
        sys.exit(1)

    target_hash = sys.argv[2]
    while True:
        s = random_string(5)
        hash_value = hashlib.md5(s.encode()).hexdigest()
        if hash_value.startswith(target_hash):
            print(f"Found match: {s}")
            sys.exit(0)

if __name__ == "__main__":
    main()
```

Moving ahead we needed to figure out how shall we login, visiting http://172.105.114.30:31337/showSignUp gave us a signup page. Reading the source we confirmed that the password
we provide is not taken into account while signing up instead it use a random uuid as password.

```python3
@app.route('/signUp',methods=['POST'])
def signUp():
    try:

        _name = request.form['inputName']
        _email = request.form['inputEmail']
        _pow_from_player = request.form['inputPoW']

         #we are in the private token sale phase so you cannot login ATM
        _password = str(uuid.uuid4())

        if verify_pow(_pow3, _pow_from_player):
            if _name and _email and _password:

                conn = mysql.connect()
                cursor = conn.cursor()       
                _hashed_password = generate_password_hash(_password)
                cursor.callproc('sp_createUser',(_name,_email,_hashed_password))
                data = cursor.fetchall()
                conn.commit()
                cursor.close()
                conn.close()

                return render_template('notification.html', noti = "Register Successfully. For the privacy of our customers, We cannot tell you that the user exists or not. Also, We are in the private token sale phase so you cannot login ATM.")
            else:
                return render_template('error.html', error='Enter the required fields homie')

        return render_template('error.html',error = "Something go wrong homie !")

    except Exception as e:

        return render_template('error.html',error = "Something go wrong homie !")
```

_password = str(uuid.uuid4()), is the line where the password was overwritten, so we needed to find a way to reset the password and figured out that there was another route, `/trigger-reset-passwd-with-username/<string:_username>`
which basically used username to reset the password.

```python3
@app.route('/trigger-reset-passwd-with-username/<string:_username>',methods=['GET'])
def userType(_username):

    try:

        _pow_from_player = request.args['inputPoW']

        if verify_pow(_pow,str(_pow_from_player)):
            for _check in app.config["LIST_USERNAME_BY_DEAULT"]:
                if _check in _username:
                    return render_template('error.html',error = "Permission Denied !!!")
            _secret_token = str(uuid.uuid4())

            conn = mysql.connect()
            cursor = conn.cursor()
            _hashed_password = generate_password_hash(_secret_token)
            cursor.callproc('sp_ResetPasswdUser',(_username, _hashed_password))
            data = cursor.fetchall()
            conn.commit()
            cursor.close()
            conn.close()

            _secret_reset_passwd_URL = url_for('gen_token',Type = _secret_token, _external=True)

            for _WAF_WTF in app.config["WAFWTF"]:
                if _WAF_WTF in _secret_reset_passwd_URL:
                    return render_template('error.html',error = 'Oops, Not Today !!!')
            _trigger_send_url = parse(_secret_reset_passwd_URL)

            return render_template('notification.html', noti = "Reset Passwd Successfully. If Your User Exist, Check it out !!!")


        else:
            return render_template('error.html',error = "Something go wrong homie !")

    except Exception as e:
        return render_template('error.html',error = "Something go wrong homie !")
```

Examining further I found out that it generated a secret token `_secret_token = str(uuid.uuid4())`, updated the users password and generated the password reset link.

`_secret_reset_passwd_URL = url_for('gen_token',Type = _secret_token, _external=True)` 

The following code was responsible for handling password reset and the step where I was stuck to figure out how will I be able to steal the secret token as we do not have control over the URL parameter.

I reached out to my friend and @fredd from THC helped me with the hint of Host header injection, and it worked. Reading the source again there was parameter `_external` set to true.

Reading the docs it was known why it worked.

`
_external (Optional[bool]) – If given, prefer the URL to be internal (False) or require it to be external (True). External URLs include the scheme and domain. When not in an active request, URLs are external by default.`

So the application would take HOST header while drafting a request and when manipulated we can get login details.

