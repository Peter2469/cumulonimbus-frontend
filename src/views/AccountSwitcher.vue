<template>
  <h1>Switch Accounts</h1>
  <h2>Have multiple accounts you need to keep track of? You can do it here.</h2>
  <div class="quick-action-buttons-container">
    <BackButton fallback="/">Back</BackButton>
    <button
      v-if="!managingAccounts"
      @click="managingAccounts = true"
      :disabled="user.loading || !online"
    >
      Remove Accounts
    </button>
    <button v-else @click="managingAccounts = false" :disabled="user.loading">
      Done
    </button>
    <button
      v-if="managingAccounts"
      @click="removeAllAccountsModal?.show()"
      :disabled="user.loading || !online"
    >
      Remove All Accounts
    </button>
  </div>
  <EmphasizedBox :no-padding="!user.loading && online">
    <template v-if="online">
      <div v-if="!user.loading" class="account-switcher-accounts">
        <div
          :class="`account-switcher-account${user.loading ? ' disabled' : ''}`"
          v-for="(token, account) in user.accounts"
          @click="handleAccountClick(account as string)"
        >
          <img
            v-if="!managingAccounts"
            :src="profileIcon"
            alt="Generic account icon"
          />
          <img v-else :src="closeIcon" alt="Delete account icon" />
          <p v-text="account" />
          <p
            v-if="user.account?.user.username === account && !managingAccounts"
          >
            Currently logged in.
          </p>
          <p v-if="token === false && !managingAccounts">Not logged in.</p>
          <p v-if="managingAccounts">Click to remove this account.</p>
        </div>
        <div
          v-if="!managingAccounts"
          :class="`account-switcher-account${user.loading ? ' disabled' : ''}`"
          @click="addAccount"
        >
          <img :src="plusIcon" alt="Plus icon" />
          <p>Add Account</p>
          <p>Add another account.</p>
        </div>
      </div>
      <LoadingBlurb v-else />
    </template>
    <template v-else>
      <p>You are offline.</p>
    </template>
  </EmphasizedBox>
  <ConfirmModal
    title="Remove All Accounts"
    ref="removeAllAccountsModal"
    @submit="removeAllAccountsConfirm"
  >
    <p>Are you sure you want to remove all accounts?</p>
    <p>You'll have to enter your username/email and password again.</p>
  </ConfirmModal>
  <ConfirmModal
    title="Remove Account"
    ref="removeAccountModal"
    @submit="removeAccountConfirm"
  >
    <p>
      Are you sure you want to remove the account
      <code v-text="selectedAccount" />?
    </p>
    <p>You'll have to enter your username/email and password again.</p>
  </ConfirmModal>
</template>

<script lang="ts" setup>
import BackButton from "@/components/BackButton.vue";
import EmphasizedBox from "@/components/EmphasizedBox.vue";
import ConfirmModal from "@/components/ConfirmModal.vue";
import Cumulonimbus from "cumulonimbus-wrapper";
import LoadingBlurb from "@/components/LoadingBlurb.vue";
import { ref, onBeforeMount, computed } from "vue";
import { userStore, cumulonimbusOptions } from "@/stores/user";
import { useRouter, useRoute } from "vue-router";
import { toastStore } from "@/stores/toast";
import profileIcon from "@/assets/images/profile.svg";
import plusIcon from "@/assets/images/plus.svg";
import closeIcon from "@/assets/images/close.svg";
import defaultErrorHandler from "@/utils/defaultErrorHandler";
import { useOnline } from "@vueuse/core";

const user = userStore(),
  router = useRouter(),
  route = useRoute(),
  toast = toastStore(),
  online = useOnline(),
  managingAccounts = ref(false),
  selectedAccount = ref<string | null>(null),
  removeAllAccountsModal = ref<typeof ConfirmModal>(),
  removeAccountModal = ref<typeof ConfirmModal>(),
  redirectLoc = computed(() =>
    route.query.redirect ? (route.query.redirect as string) : "/dashboard"
  );

async function redirect() {
  await router.replace(redirectLoc.value);
}

async function handleAccountClick(account: string) {
  if (user.loading) return;
  if (managingAccounts.value) {
    selectedAccount.value = account;
    removeAccountModal.value?.show();
  } else {
    try {
      const res = await user.switchAccount(account);
      if (typeof res === "boolean") {
        if (res) {
          toast.show(`Switched to ${user.account?.user.username}.`);

          redirect();
        } else
          router.replace({
            path: "/auth",
            query: {
              redirect: redirectLoc.value,
              username: account,
            },
            hash: "#login",
          });
      } else {
        toast.show(`Failed to switch to ${account}: ${res.message}`);
      }
    } catch (e) {
      if (e instanceof Cumulonimbus.ResponseError) {
        const handled = await defaultErrorHandler(e, router);
        if (!handled) toast.show(`Failed to switch to ${account}.`);
      } else {
        toast.clientError();
      }
    }
  }
}

function addAccount() {
  router.replace({
    path: "/auth",
    query: {
      redirect: redirectLoc.value,
    },
    hash: "#login",
  });
}

async function removeAllAccountsConfirm(choice: boolean) {
  if (choice) {
    for (const account in user.accounts) {
      if (user.accounts[account] === false) {
        user.removeAccount(account);
      } else {
        const tempClient = new Cumulonimbus(
          user.accounts[account] as string,
          cumulonimbusOptions
        );
        try {
          const res = await tempClient.deleteSelfSession(
            (await tempClient.getSelfSession()).result.iat.toString()
          );
          if (res) {
            user.removeAccount(account);
          }
        } catch (error) {
          if (error instanceof Cumulonimbus.ResponseError) {
            if (error.code === "INVALID_SESSION_ERROR") {
              user.removeAccount(account);
            }
            {
              console.error(error);
              toast.clientError();
            }
          } else {
            console.error(error);
            toast.clientError();
          }
        }
      }
    }
    toast.show("All accounts removed.");
    user.logout();
    router.replace({
      path: "/auth",
      query: {
        redirect: redirectLoc.value,
      },
      hash: "#login",
    });
  }
}

async function removeAccountConfirm(choice: boolean) {
  if (choice) {
    if (user.accounts[selectedAccount.value as string] === false) {
      toast.show("Account removed.");
      user.removeAccount(selectedAccount.value as string);
      selectedAccount.value = null;
      removeAccountModal.value?.hide();
    } else {
      const tempClient = new Cumulonimbus(
        user.accounts[selectedAccount.value as string] as string,
        cumulonimbusOptions
      );
      try {
        const res = await tempClient.deleteSelfSession(
          (await tempClient.getSelfSession()).result.iat.toString()
        );
        if (res) {
          toast.show("Account removed.");
          if (user.account?.user.username === selectedAccount.value) {
            user.logout();
          }
          user.removeAccount(selectedAccount.value as string);
        }
      } catch (error) {
        if (error instanceof Cumulonimbus.ResponseError) {
          if (error.code === "INVALID_SESSION_ERROR") {
            toast.show("Account removed.");
            if (user.account?.user.username === selectedAccount.value) {
              user.logout();
            }
            user.removeAccount(selectedAccount.value as string);
          } else {
            console.error(error);
            toast.clientError();
          }
        } else {
          console.error(error);
          toast.clientError();
        }
      } finally {
        removeAccountModal.value?.hide();
      }
    }
  } else {
    removeAccountModal.value?.hide();
  }
}

onBeforeMount(async () => {
  if (Object.keys(user.accounts).length < 1) {
    router.replace({
      path: "/auth",
      query: {
        redirect: redirectLoc.value,
      },
      hash: "#login",
    });
  } else if (Object.keys(user.accounts).length < 2 && !user.loggedIn) {
    const res = await user.switchAccount(Object.keys(user.accounts)[0]);
    if (typeof res === "boolean") {
      if (res)
        router.replace({
          path: redirectLoc.value,
        });
      else
        router.replace({
          path: "/auth",
          query: {
            redirect: redirectLoc.value,
            username: Object.keys(user.accounts)[0],
          },
          hash: "#login",
        });
    }
  }
});
</script>

<style>
.account-switcher-accounts {
  width: fit-content;
  display: flex;
  flex-wrap: nowrap;
  justify-content: center;
  flex-direction: column;
}

.account-switcher-account {
  display: grid;
  grid: 1fr 1fr / 50px 4fr;
  align-items: center;
  padding: 25px 30px;
  gap: 0 5px;
  border-bottom: 1px solid var(--ui-border);
  cursor: pointer;
  transition: background-color 0.25s, border-color 0.25s, color 0.25s;
}

.account-switcher-account:hover,
.account-switcher-account:focus {
  background-color: var(--ui-background-hover);
}

.account-switcher-account:first-child {
  border-radius: 1rem 1rem 0 0;
}

.account-switcher-account:last-child {
  border-radius: 0 0 1rem 1rem;
  border-bottom-width: none;
  border-bottom-style: none;
}

.account-switcher-account:only-child {
  border-radius: 1rem;
  border-bottom-width: none;
  border-bottom-style: none;
}

.account-switcher-account.disabled {
  cursor: not-allowed;
  background-color: var(--ui-background-disabled);
  color: var(--ui-foreground-disabled);
}

.account-switcher-account img {
  transition: filter 0.25s;
  width: 50px;
  justify-self: left;
  align-self: center;
  grid-row: 1 / span 2;
  grid-column: 1 / 2;
  user-select: none;
  -moz-user-select: none;
  -webkit-user-select: none;
}

html.dark-theme .account-switcher-account img {
  filter: invert(100%);
}

html.dark-theme .account-switcher-account.disabled img {
  filter: invert(calc(148 / 255));
}

.account-switcher-account p {
  margin: 0;
  text-align: left;
  transition: color 0.25s;
}

.account-switcher-account.disabled p {
  color: var(--ui-foreground-disabled);
}

.account-switcher-account p:first-of-type {
  font-size: 24px;
  font-weight: bold;
  font-family: var(--font-heading);
  grid-column: 2 / 2;
  grid-row: 1 / 2;
}

.account-switcher-account p:first-of-type:not(:only-of-type) {
  align-self: end;
}

.account-switcher-account p:last-of-type:not(:only-of-type) {
  align-self: start;
  grid-column: 2 / 2;
  grid-row: 2 / 2;
}

.account-switcher-account p:only-of-type {
  grid-row: 1 / span 2;
}
</style>
