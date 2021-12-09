<template>
  <!-- Vuetify App Component -->
  <v-app>
    <!-- App Bar: Holds the title and login/save buttons -->
    <v-app-bar
      app
      color="green darken-2"
      dark
    >
      <v-app-bar-title>Todo App</v-app-bar-title>
      <!-- v-spacer expands to fill empty space, pushing elements outside it to either side -->
      <v-spacer />
      <!-- v-if is used here to only show buttons when they are relevant -->
      <!-- @click defines what method is called when the button is clicked -->
      <!-- :loading creates a one-way binding for the loading attribute from the saving property -->
      <v-btn
        v-if="user !== null && changedTasks"
        text
        :loading="saving"
        @click="saveTasks"
      >
        Save
      </v-btn>

      <v-btn
        v-if="user === null"
        @click="signIn"
      >
        Sign in
      </v-btn>

      <v-btn
        v-else
        @click="signOut"
      >
        Sign out
      </v-btn>
    </v-app-bar>

    <!-- App Content: Holds the tasks and the add task form --> -->
    <v-main>
      <!-- Container just sets a max width for the content so it's easier on the eyes -->
      <v-container>
        <!-- Show a loading bar if firebase hasn't loaded the user,
          or if we are signed in and the data hasn't loaded yet -->
        <v-progress-linear
          v-if="!userLoaded || (user && !dataLoaded)"
          indeterminate
        />
        <!-- If the above conditions for loading aren't met, show the app. -->
        <v-card v-else>
          <v-card-title>
            To-Do List
          </v-card-title>
          <v-card-text>
            <v-list>
              <!-- v-for is used here to iterate over the tasks array -->
              <v-list-item
                v-for="item in tasks"
                :key="item.task"
              >
                <!-- Bind a checkbox to the complete property of each task -->
                <v-simple-checkbox
                  v-model="item.complete"
                  @input="sortTasks"
                />
                <!-- Bind the task property of each task to the text of the list item -->
                <!-- Also bind the complete class to the value of the item's complete property -->
                <span :class="{complete: item.complete}">{{ item.task }}</span>
                <v-spacer />
                <!-- Run the deleteTask method when the delete button is pressed,
                  passing the specific task name to the method as an argument -->
                <v-btn
                  icon
                  @click="removeTask(item.task)"
                >
                  <v-icon>mdi-trash-can</v-icon>
                </v-btn>
              </v-list-item>
            </v-list>
          </v-card-text>
          <v-card-actions>
            <!-- Create a two-way binding between this field and the newTask variable,
              and run addTask when enter is pressed -->
            <v-text-field
              v-model="newTask"
              label="Add Task"
              outlined
              @keydown.enter="addTask()"
            />
          </v-card-actions>
        </v-card>
      </v-container>
    </v-main>
  </v-app>
</template>

<script>
// Import all of our npm modules that we need to use.
// This code wouldn't normally work in the browser,
//  but we can use it here because webpack is bundling our dependencies for us.
import Vue from 'vue';
import { initializeApp } from 'firebase/app';
import { getAnalytics } from 'firebase/analytics';
import {
  getAuth, GoogleAuthProvider, onAuthStateChanged, signInWithPopup,
} from 'firebase/auth';
import {
  doc, getFirestore, setDoc, getDoc, onSnapshot,
} from 'firebase/firestore';
import { isEqual, cloneDeep } from 'lodash';

// Define the Vue component's functionality.
export default {
  // Component name, isn't really important here,
  //  but is used by Vue to identify the component if you have multiple or use it elsewhere.
  name: 'App',

  // The data property is where we define the data that the component will use.
  // It's an object with properties that are accessible in the template.
  // Note that we need to use a function to define the data, due to the way Vue works.
  // (See https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function)
  data: () => ({
    tasks: [],
    cloudTasks: [],
    newTask: '',
    user: null,
    userLoaded: false,
    dataLoaded: false,
    saving: false,
  }),

  // Computed properties are used to define methods that can be used in the template as values.
  // These are run when the component is first created, and then whenever the input data changes.
  // This means that repeated calls to this value will just use the cached value,
  //  unless the data changes.
  computed: {
    changedTasks() {
      return !isEqual(this.tasks, this.cloudTasks);
    },
  },

  // This method is called when the component is created, but before the template is rendered.
  // i.e. after the JS is loaded, but before the HTML is loaded.
  // It's useful for setting up the initial state of the component.
  created() {
    // Your web app's Firebase configuration
    // For Firebase JS SDK v7.20.0 and later, measurementId is optional
    // You can just cut/paste this from the firebase console.
    const firebaseConfig = {
      apiKey: 'AIzaSyAN6ulhhv8wO3glvMe5zg2R91QLBVoAPqE',
      authDomain: 'fsyc-todo-app.firebaseapp.com',
      projectId: 'fsyc-todo-app',
      storageBucket: 'fsyc-todo-app.appspot.com',
      messagingSenderId: '826636038235',
      appId: '1:826636038235:web:8f8e5eb4c67a3ac9840f7e',
      measurementId: 'G-N48Y7P1ZMB',
    };

    // Initialize Firebase
    const app = initializeApp(firebaseConfig);
    // Start up the analytics module to get usage data.
    getAnalytics(app);

    // Get the auth module from firebase.
    const auth = getAuth();
    // Watch for changes to the user's authentication state,
    //  and update the user property accordingly.
    onAuthStateChanged(auth, (user) => {
      // Mark the user as loaded.
      this.userLoaded = true;

      // Set the user property.
      this.user = user;

      // If the user is logged in, load the data.
      if (user) {
        this.getTasks();
      }
    });
  },

  // Methods are used to define functionality that can be used in the template.
  // They act like a function, but are called with `this` set to the component.
  // Note: You cannot use arrow functions here, because they don't bind `this`.
  methods: {
    // This method is called when the user clicks the "Add Task" button.
    // It adds the new task to the list of tasks, and clears the input field.
    addTask() {
      // Check if a task already exists with the same name.
      if (this.tasks.every((task) => task.task !== this.newTask)) {
        // If not, add the new task to the list of tasks.
        this.tasks.push({ task: this.newTask, complete: false });
        // Clear the input field.
        this.newTask = '';
        // Sort the tasks by completion status, and then name.
        this.sortTasks();
      } else {
        // If a task with the same name already exists, alert the user.
        // nb: Using alert is generally considered bad practice, as it blocks the main thread.
        //  but it's fine here because this is a demo app.
        //  Don't do this in your real app!
        //  As an alternative, you could use Vuetify's snackbar component, or a dialog.
        //  Remember: Do as I say, not as I do.
        alert('Task already exists');
      }
    },

    // This method is called when the user clicks the "Delete" button.
    // It accepts the name of a task, and removes it from the list of tasks.
    removeTask(taskName) {
      // Find the index of the first task with the given name.
      const taskIdx = this.tasks.findIndex((task) => task.task === taskName);
      // If the task was found, remove it from the list of tasks.
      if (taskIdx >= 0) {
        // We need to use Vue.delete() to remove the task from the list.
        // This is because in vue 2 some things don't trigger reactive updates.
        // I forget which things don't trigger reactive updates, so I just use Vue.delete()
        // and Vue.set() all the time. You don't have to do this, but it can't hurt.
        Vue.delete(this.tasks, taskIdx);
        // Sort the tasks by completion status, and then name.
        this.sortTasks();
      }
    },

    // This method sorts the tasks by completion status, and then name.
    sortTasks() {
      this.tasks.sort((a, b) => {
        // If the tasks are both complete or both incomplete, sort by name.
        if (a.complete === b.complete) {
          return a.task.localeCompare(b.task);
        }
        // If one task is complete and the other is not, sort by completion status.
        return a.complete ? 1 : -1;
      });
    },

    // This method is called when the user clicks the "Sign-in" button.
    signIn() {
      // Get the auth module from firebase.
      const auth = getAuth();
      // Get the Google auth provider from firebase.
      const provider = new GoogleAuthProvider();

      // Sign in with Google.
      signInWithPopup(auth, provider)
        .catch((error) => {
          // If there was an error, write it in the console.
          console.error(error.message);
        });
    },

    // This method is called when the user clicks the "Sign-out" button.
    signOut() {
      // Get the auth module from firebase.
      const auth = getAuth();
      // Sign out.
      auth.signOut();
    },

    // This method gets the tasks from firestore.
    async getTasks() {
      // Get firestore.
      const db = getFirestore();
      // Get the current user.
      // This is equivalent to const user = this.user;
      // (See https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)
      const { user } = this;
      // If the user isn't signed in, exit the function.
      if (user === null) return;
      // Define a function to save the tasks once they're retrieved.
      const handleData = (docSnapshot) => {
        // If the document exists, save the tasks.
        if (docSnapshot.exists()) {
          // Set the tasks to the tasks in the document.
          this.tasks = docSnapshot.data().tasks;
          // Set the cloudTasks to a copy of the tasks.
          this.cloudTasks = cloneDeep(this.tasks);
          // Sort the tasks.
          this.sortTasks();
        } else {
          // If the document doesn't exist, put a warning in the console.
          console.warn('No tasks found for user');
        }
      };

      // Whenever the document changes in firestore, call handleData.
      onSnapshot(doc(db, 'users', user.uid), handleData);

      // Get the current document from firestore and call handleData.
      handleData(await getDoc(doc(db, 'users', user.uid)));

      this.dataLoaded = true;
    },

    // This method is called when the user clicks the "Save" button.
    async saveTasks() {
      // Set saving to true.
      this.saving = true;

      // Get firestore.
      const db = getFirestore();
      // Get the current user.
      const { user } = this;
      // If the user isn't signed in, exit the function.
      if (user === null) {
        this.saving = false;
        return;
      }

      // Set the user's tasks in firestore.
      await setDoc(doc(db, 'users', user.uid), { tasks: this.tasks });

      // Set cloudTasks to a copy of the tasks.
      this.cloudTasks = cloneDeep(this.tasks);

      // Set saving to false.
      this.saving = false;
    },
  },
};
</script>

<style>
    .complete {
      text-decoration: line-through;
      color: grey;
    }
</style>
