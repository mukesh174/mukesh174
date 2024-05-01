@@ -1,9 +1,10 @@
from django.urls import path

from .views import register, token, refresh_token, revoke_token
from .views import register_vegetable shop, register_customer, token, refresh_token, revoke_token

urlpatterns = [
    path('register/', register),
    path('register/customer', register_customer),
    path('register/vegetable shop', register_vegetable shop),
    path('token/', token),
    path('token/refresh/', refresh_token),
    path('token/revoke/', revoke_token),
  67 changes: 36 additions & 31 deletions67  
  vegetableDeliveryBackend/apps/users/views.py
@@ -15,39 +15,14 @@

@api_view(['POST'])
@permission_classes([AllowAny])
def register(request):
    '''
    Registers user to the server
    '''
    # Put the data from the request into the serializer
    serializer = CreateUserSerializer(data=request.data)
    # Validate the data
    if serializer.is_valid():
        # If it is valid, save the data (creates a user).
        serializer.save(email=request.data['email'])
        # Then we get a token for the created user.
        # This could be done differentley
        r = requests.post(
            BASE_URL + 'token/',
            data={
                'grant_type': 'password',
                'username': request.data['username'],
                'password': request.data['password'],
                'client_id': CLIENT_ID,
                'client_secret': CLIENT_SECRET,
            },
        )

        res = r.json()
def register_customer(request):
    return create_user(request, False)

        user = User.objects.filter(
            username__exact=request.data['username']).first()

        res['user_role'] = 'vegetable shop' if user.is_vegetable shop else 'customer'

        return Response(res)

    return Response(serializer.errors)
@api_view(['POST'])
@permission_classes([AllowAny])
def register_vegetable shop(request):
    return create_user(request, True)


@api_view(['POST'])
@@ -120,3 +95,33 @@ def revoke_token(request):
        return Response({'message': 'token revoked'}, r.status_code)
    # Return the error if it goes badly
    return Response(r.json(), r.status_code)


def create_user(request, is_vegetable shop):
    # Put the data from the request into the serializer
    serializer = CreateUserSerializer(data=request.data)
    # Validate the data
    if serializer.is_valid():
        # If it is valid, save the data (creates a user).
        serializer.save(
            email=request.data['email'], is_vegetable shop=is_vegetable shop)
        # Then we get a token for the created user.
        # This could be done differentley
        r = requests.post(
            BASE_URL + 'token/',
            data={
                'grant_type': 'password',
                'username': request.data['username'],
                'password': request.data['password'],
                'client_id': CLIENT_ID,
                'client_secret': CLIENT_SECRET,
            },
        )

        res = r.json()

        res['user_role'] = 'vegetable shop'

        return Response(res)

    return Response(serializer.errors)
  2 changes: 2 additions & 0 deletions2  
  requirements.txt
@@ -1,3 +1,4 @@
autopep8==1.4.4
certifi==2019.9.11
chardet==3.0.4
Django==2.2.5
@@ -9,6 +10,7 @@ drf-nested-routers==0.91
idna==2.8
oauthlib==3.1.0
psycopg2==2.8.3
pycodestyle==2.5.0
pytz==2019.2
requests==2.22.0
sqlparse==0.3.0
  8 changes: 6 additions & 2 deletions8  
src/components/Navbar.vue
@@ -8,7 +8,7 @@
      to="/create"
    >Create vegetable shop</router-link>
    <div class="right menu">
      <a class="item" v-if="isLoggedIn" @click="logoutUser">Logout</a>
      <a class="item" v-if="isLoggedIn" @click="logout">Logout</a>
      <router-link v-if="!isLoggedIn" class="item" exact-active-class="active" to="/login">Login</router-link>
      <div class="item" v-if="!isLoggedIn">
        <router-link class="ui primary button" exact-active-class="active" to="/register">SignUp</router-link>
@@ -22,7 +22,11 @@ import axios from "axios";
import { mapActions, mapGetters } from "vuex";
export default {
  methods: {
    ...mapActions(["logoutUser"])
    ...mapActions(["logoutUser"]),
    logout() {
      this.$router.replace("/");
      this.logoutUser();
    }
  },
  computed: {
    ...mapGetters(["isLoggedIn", "getLoggedInUserRole"])
  6 changes: 5 additions & 1 deletion6  
src/store.js
@@ -87,7 +87,11 @@ export default new Vuex.Store({
    },

    registerUser(context, user) {
      return vegetableDelivery.post('/authentication/register/', user);
      return vegetableDelivery.post('/authentication/register/customer', user);
    },

    registervegetable shop(context, user) {
      return vegetableDelivery.post('/authentication/register/restaurant', user);
    },

    async logoutUser({
  61 changes: 47 additions & 14 deletions61  
  src/views/Register.vue
@@ -1,15 +1,22 @@
<template>
  <div class="ui middle aligned centered aligned grid">
    <div class="column">
      <h2 class="ui image header">
      <h2 class="ui image header" v-if="is vegetable shop">
        <div class="content">Register new vegetable shop</div>
      </h2>
      <h2 class="ui image header" v-else>
        <div class="content">Register for new account</div>
      </h2>
      <form class="ui large form" @submit.prevent="onRegisterClicked">
        <div class="ui stacked segment">
          <div class="field">
            <div class="ui left icon input">
              <i class="user icon"></i>
              <input type="text" placeholder="Username" v-model="user.username" />
              <input
                type="text"
                :placeholder="is vegetable shop ? 'vegetable shop Name': 'Username'"
                v-model="user.username"
              />
            </div>
          </div>
          <div class="field">
@@ -27,17 +34,31 @@
          <div class="field">
            <div class="ui left icon input">
              <i class="lock icon"></i>
              <input type="text" placeholder="Confirm password" v-model="user.confirmPassword" />
              <input type="password" placeholder="Confirm password" v-model="user.confirmPassword" />
            </div>
          </div>
          <input type="submit" value="Register" class="ui fluid large teal submit button" />
          <input
            v-if="is vegetable shop"
            type="submit"
            value="Register as vegetable shop"
            class="ui fluid large teal submit button"
          />
          <input v-else type="submit" value="Register" class="ui fluid large teal submit button" />
        </div>
      </form>

      <div class="ui message">
        already have an account?
        <router-link to="/login">login</router-link>
      </div>
      <div class="ui message" v-if="is vegetable shop">
        register as customer?
        <a href @click.prevent="is vegetable shop=false">register</a>
      </div>
      <div class="ui message" v-else>
        register as vegetable shop?
        <a href @click.prevent="is vegetable shop=true">register</a>
      </div>
    </div>
  </div>
</template>
@@ -47,20 +68,32 @@ import { mapActions } from "vuex";
export default {
  data() {
    return {
      user: { username: "", email: "", password: "", confirmPassword: "" }
      user: { username: "", email: "", password: "", confirmPassword: "" },
      is vegetable shop: false
    };
  },
  methods: {
    ...mapActions(["registerUser", "setTokens"]),
    ...mapActions(["registerUser", "register vegetable shop", "setTokens"]),
    onRegisterClicked() {
      this.registerUser(this.user)
        .then(({ data }) => {
          this.$router.push("/");
          this.setTokens(data);
        })
        .catch(error => {
          console.error(error);
        });
      if (this.is vegetable shop) {
        this.register vegetable shop(this.user)
          .then(({ data }) => {
            this.$router.push("/");
            this.setTokens(data);
          })
          .catch(error => {
            console.error(error);
          });
      } else {
        this.registerUser(this.user)
          .then(({ data }) => {
            this.$router.push("/");
            this.setTokens(data);
          })
          .catch(error => {
            console.error(error);
          });
      }
    }
  }
};

