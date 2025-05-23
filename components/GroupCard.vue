<template>
  <div class="p-4 border shadow-sm bg-white rounded-xl w-full">
    <img
      :src="group.groupUrl || '/images/group.png'"
      alt="Group Image"
      class="w-full h-32 object-cover rounded-t-lg mb-2"
    />
    <div class="flex items-center justify-between">
      <h3 class="text-lg font-semibold text-gray-900">{{ group.name }}</h3>
      <div
        class="flex items-center bg-blue-100 text-blue-700 text-xs font-medium px-2 py-1 rounded-full"
      >
        <span class="mr-1">👥</span>
        {{ group.members }}
      </div>
    </div>

    <p class="text-gray-600 text-sm mt-3">{{ group.desc }}</p>

    <button
      @click="joinGroup"
      :disabled="isJoining || isMember"
      class="mt-4 w-full flex items-center justify-center gap-2 py-2 bg-blue-600 text-white rounded-lg font-medium hover:bg-blue-700 transition disabled:bg-gray-300 disabled:cursor-not-allowed"
    >
      <Icon icon="mdi:plus" />
      {{
        isMember ? "Already Joined" : isJoining ? "Joining..." : "Join Group"
      }}
    </button>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import { getAuth } from "firebase/auth";
import { ref as dbRef, set, get } from "firebase/database";
import { Icon } from "@iconify/vue";
import { auth, db } from "~/firebase/firebase.js";

const props = defineProps(["group"]);

const isJoining = ref(false);
const isMember = ref(false);

const checkMembership = async () => {
  const user = auth.currentUser;
  if (!user) return false;

  try {
    const memberRef = dbRef(db, `chatroom_users/${props.group.id}/${user.uid}`);
    const snapshot = await get(memberRef);
    isMember.value = snapshot.exists();
  } catch (error) {
    console.error("Error checking membership:", error);
  }
};

checkMembership();

// Join group function
const joinGroup = async () => {
  const user = auth.currentUser;
  if (!user || isMember.value) return;

  isJoining.value = true;

  try {
    const joinedAt = Date.now();
    // 1.Add user to group member list
    const memberRef = dbRef(db, `chatroom_users/${props.group.id}/${user.uid}`);
    await set(memberRef, {
      role: "user",
      joinedAt: joinedAt,
      isPinned: false,
      isMuted: false,
    });

    // 2.Add a group to the user's group list
    const userGroupRef = dbRef(
      db,
      `user_chatrooms/${user.uid}/${props.group.id}`
    );
    await set(userGroupRef, true);

    // 3. Update local status
    isMember.value = true;
    props.group.members += 1;

    await writeActivityLog(
      props.group.id,
      auth.currentUser?.uid,
      `${await getUsername(auth.currentUser?.uid)} has joined the group`,
      joinedAt + 1
    );

    //await updateGroupKey(props.group.id);
  } catch (error) {
    console.error("Error joining group:", error);
  } finally {
    isJoining.value = false;
  }
};
</script>
