<template>
  <StackLayout ref="mainContainer" class="main-container">
<!--    <Image src="~/assets/images/fpr-logo-full.png" width="200" @tap="quickLogin"></Image>-->
    <Image src="~/assets/images/fpr-logo-full.png" width="200"></Image>

    <GridLayout ref="formControls" class="form-controls" rows="auto, auto">
      <TextField
          hint="E-mail adres"
          keyboardType="email"
          returnKeyType="next"
          @returnPress="focusPassword()"
          v-model="user.email"
          autocorrect="false"
          autocapitalizationType="none"
          row="0"></TextField>

      <TextField ref="password"
                 hint="Wachtwoord"
                 secure="true"
                 returnKeyType="done"
                 @returnPress="login()"
                 v-model="user.password"
                 row="1"></TextField>

      <ActivityIndicator :busy="isAuthenticating" rowSpan="2" width="30" height="30"></ActivityIndicator>
    </GridLayout>

    <Button
        text="AANMELDEN"
        :isEnabled="!isAuthenticating"
        class="btn btn-secondary"
        @tap="login"></Button>

    <Label
        class="forgot-password"
        text="Wachtwoord vergeten?"
        @tap="forgotPassword()"></Label>

    <Label :text="appVersion" class="app-version"></Label>

    <StackLayout ref="signUpStack" class="sign-up-stack" @tap="toggleDisplay()" translateY="50">
      <Label :text="isLoggingIn ? 'Sign up here' : 'Back to login'"></Label>
    </StackLayout>
  </StackLayout>
</template>

<script lang="ts">
  import * as AppVersion from "nativescript-appversion";
  import { connectionType, getConnectionType } from "tns-core-modules/connectivity"
  import { device } from "tns-core-modules/platform";
  import { action, prompt } from "tns-core-modules/ui/dialogs"
  import { applicationSettingsService } from "~/main";
  import routes from "~/router";
  import alert from "~/utils/alert"
  import { setUserId, setUserPropertyUsertype } from "~/utils/analytics-util";
  import { dismissKeyboard } from "~/utils/keyboard-util";

  export default {
    name: 'login-main',

    props: {
      visible: true
    },

    data() {
      return {
        appVersion: undefined,
        isLoggingIn: true,
        isAuthenticating: false,
        user: {
          email: applicationSettingsService.getUsername(),
          password: ""
        }
      }
    },

    mounted() {
      this.appVersion = `v ${AppVersion.getVersionNameSync()} (${AppVersion.getVersionCodeSync()})`;

      // setTimeout(() => {
      //   this.user.email = "eddyverbruggen+fprAnnette@gmail.com";
      //   this.user.password = "xs4all";
      //   this.login();
      // }, 1000);
    },

    methods: {
      quickLogin() {
        const options = [
          "Annette (trainer Hoogland JO9-7)",
          "Bas V (speler Hoogland JO9-7)"
        ];
        const cancelLabel = "Annuleren";
        action({
          title: "Kies een login..",
          actions: options,
          cancelable: true,
          cancelButtonText: cancelLabel
        }).then(picked => {
          if (picked && picked !== cancelLabel) {
            if (picked === "Mark (trainer & speler Victoria H2)") {
              this.user.email = "fennemrdejong@gmail.com";
              this.user.password = "fpr123";
            } else if (picked === "Nick (speler Victoria H2)") {
              this.user.email = "nickwagenaar9@gmail.com";
              this.user.password = "fpr123";
            } else if (picked === "Daniel (speler Victoria H2)") {
              this.user.email = "danielhindriks@gmail.com";
              this.user.password = "fpr123";
            } else if (picked === "Annette (trainer Hoogland JO9-7)") {
              this.user.email = "eddyverbruggen+fprAnnette@gmail.com";
              this.user.password = "xs4all";
            } else if (picked === "Bas V (speler Hoogland JO9-7)") {
              this.user.email = "hoogland.JO9-7.player1@fpr.com";
              this.user.password = "xs4all";
            }
            this.login();
          }
        });
      },

      toggleDisplay() {
        this.isLoggingIn = !this.isLoggingIn;
      },

      focusPassword() {
        this.$refs.password.nativeView.focus();
      },

      login(event) {
        if (getConnectionType() === connectionType.none) {
          alert("Om in te kunnen loggen is een internetverbinding vereist");
          return;
        }

        this.isAuthenticating = true;

        if (event) {
          dismissKeyboard(event.object);
        }

        return this.$authService
            .login(this.user)
            .then(() => {
              applicationSettingsService.setUsername(this.user.email);
              this.isAuthenticating = false;
              this.$editingUserService.userWrapper.team = undefined;
              this.$editingUserService.userWrapper.user = this.$authService.userWrapper.user;
              this.$editingUserService.watchUser();
              this.$navigateTo(routes.home, {clearHistory: true});
              this.visible = false;
              setUserId(this.$authService.userWrapper.user.id);
              if (this.$authService.userWrapper.user.trains) {
                setUserPropertyUsertype("trainer");
              }
              if (this.$authService.userWrapper.user.playsin) {
                setUserPropertyUsertype("player");
              }
              // set the current app version for this user - may come in handy when helping users
              this.$editingUserService.updateUserDataInFirebase({
                appinfo: {
                  appversion: this.appVersion,
                  os: device.os.toLowerCase(),
                  osversion: device.osVersion,
                  logins: this.$authService.userWrapper.user.appinfo !== undefined && this.$authService.userWrapper.user.appinfo.logins !== undefined ? this.$authService.userWrapper.user.appinfo.logins + 1 : 1
                }
              }).then(() => console.log(`Set appVersion to ${this.appVersion} for user`));
            })
            .catch(error => {
              this.isAuthenticating = false;
              console.error(error);
              alert("Dat lijkt niet te kloppen.\nProbeer het nog eens.");
              /*
            if (error.includes("The password is invalid")) {
              // user found
              alert("Dat wachtwoord lijkt niet te kloppen.\nProbeer het nog eens.");
            } else {
              // no user found
              prompt({
                title: "Bevestig je e-mail adres",
                defaultText: this.user.email,
                okButtonText: "Verder >",
                cancelButtonText: "Annuleren"
              }).then(data => {
                if (data.result && data.text && data.text.trim().length > 4) {
                  this.user.email = data.text.trim();
                  prompt({
                    title: `Bevestig je wachtwoord`,
                    message: `Voor je nieuwe account\n${this.user.email}`,
                    defaultText: this.user.password,
                    okButtonText: "Registreren",
                    cancelButtonText: "Annuleren"
                  }).then(data => {
                    if (data.result && data.text && data.text.trim().length > 0) {
                      console.log("data.result: " + JSON.stringify(data));
                      this.user.password = data.text.trim();
                      this.signUp();
                    }
                  });
                }
              });
            }
             */
            });
      },

      // TODO perhaps it's better for now to not allow users registering
      signUp() {
        if (getConnectionType() === connectionType.none) {
          alert("Om te kunnen registreren is een Internetverbinding vereist")
          return;
        }
        this.$authService
            .register(this.user.email, this.user.password)
            .then(() => {
              alert("Je account is succesvol aangemaakt. Welkom!");
              this.login();
            })
            .catch(error => alert(error));
      },

      forgotPassword() {
        prompt({
          title: "Wachtwoord vergeten",
          message: "Voer je registratie e-mail in, zodat we je wachtwoord kunnen resetten",
          defaultText: "",
          okButtonText: "Reset!",
          cancelButtonText: "Annuleren"
        }).then(data => {
          if (data.result) {
            this.$authService
                .resetPassword(data.text.trim())
                .then(() => {
                  alert("Je wachtwoord is succesvol gereset. We hebben je een e-mail gestuurd met verdere instructies.");
                })
                .catch(error => {
                  console.log('Error resetting password: ' + error);
                  alert("Helaas is er een fout opgetreden tijdens het resetten van je wachtwoord: " + error);
                })
          }
        });
      }
    }
  }
</script>

<style lang="scss" scoped>
  .login {
    .app-version {
      color: #d6d6d6;
      font-size: 11;
      horizontal-align: center;
      margin-top: 4;
    }

    .main-container {
      width: 300;
      height: 470;
      margin-left: 30;
      margin-right: 30;
      border-radius: 10;
    }

    .main-label {
      horizontal-align: center;
      color: #131426;
    }

    .form-controls,
    .sign-up-stack {
      /*opacity: 0;*/
    }

    Image {
      margin-top: 5;
      margin-bottom: 20;
    }

    Button,
    TextField {
      margin-left: 16;
      margin-right: 16;
      margin-bottom: 10;
      border-radius: 2;
    }

    TextField {
      color: #131426;
      placeholder-color: #ACA6A7;
      margin-bottom: 10;

      &.light {
        color: #C4AFB4;
        placeholder-color: #C4AFB4;
      }
    }

    .submit-button {
      background-color: #F5D1E9;
      color: #131426;
      margin-top: 15;
      border-radius: 10;
    }

    .forgot-password {
      font-size: 13;
      color: #888888;
      text-align: center;
      padding: 6;
    }

    .sign-up-stack {
      background-color: #66A59A;

      label {
        width: 100%;
        color: white;
        horizontal-align: center;
        text-align: center;
        height: 50;
      }
    }
  }

  .platform-ios .login {
    .main-label {
      color: #131426;
      font-size: 32;
      margin-top: 45;
      margin-bottom: 52;
      letter-spacing: 0.2;
    }

    TextField {
      /*border-width: 1;*/
      /*border-color: #131426;*/
      margin-bottom: 20;

      &.light {
        /*border-color: #C4AFB4;*/
      }
    }

    .submit-button {
      height: 40;
    }
  }

  .platform-android .login {
    .main-label {
      font-size: 28;
      margin-top: 42;
      margin-bottom: 32;
      letter-spacing: 0.3;
    }

    .sign-up-stack label {
      margin-top: 15;
      text-transform: uppercase;
    }

    .forgot-password {
      font-size: 13;
      text-transform: uppercase;
    }
  }
</style>
