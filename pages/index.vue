<template>
  <v-row justify="center" align="center">
    <v-col cols="12">
      <v-card>
        <v-card-title>トークン (TK)</v-card-title>
        <v-card-subtitle>
          {{ token }}
        </v-card-subtitle>
        <v-card-text>
          <div v-if="!token">
            <v-text-field v-model="loginData.uname" outlined label="Moodleユーザー名" color="green" />
            <v-text-field outlined v-model="loginData.pwd" color="green" :append-icon="show ? 'mdi-eye' : 'mdi-eye-off'"  type="password" label="Moodleパスワード"/>

            <v-divider class="mb-3" />
            <v-btn large color="green" @click="getMoodleToken">トークンの生成</v-btn>
          </div>
          <div v-else>
            <v-btn large color="red" @click="revokeAccess">アクセス権の剥奪</v-btn>
          </div>
        </v-card-text>
      </v-card>
    </v-col>
    <v-col cols="12" sm="12" md="6" v-if="token">
      <v-card>
        <v-card-title>
          コース一覧 (CL)
        </v-card-title>
        <v-card-text>
          <v-autocomplete outlined :items="courses" v-model="selectedCourse" item-text="apifullname" item-value="ik" color="green" item-color="green"/>
        </v-card-text>
      </v-card>

      <v-card v-if="selectedCourse !== 0">
        <v-card-title>
          グループ一覧 ({{ courses[selectedCourse]['apifullname'] || '' }})
        </v-card-title>
        <v-card-text>
          <v-autocomplete outlined :items="groups" v-model="selectedGroup" item-text="name" item-value="ik" color="green" item-color="green"/>
        </v-card-text>
        <v-card-actions>
          <v-btn color="green" large @click="add_group_dialog = true">グループの追加</v-btn>
          <v-spacer />

          <v-btn color="red" large :disabled="selectedGroup === 0" @click="removeGroup">グループ削除</v-btn>
        </v-card-actions>
      </v-card>
    </v-col>
    <v-col cols="12" sm="12" md="6" v-if="selectedGroup !== 0">
        <v-card>
          <v-card-title>{{ groups[selectedGroup]['name'] }}のメンバー</v-card-title>
          <v-card-subtitle><v-btn color="green" @click="openAddMemberDialog">グループへのメンバー追加</v-btn></v-card-subtitle>
          <v-card-text class="mt-3">
            <v-list>
              <v-list-item v-for="(data, i) in members" :key="i">

                <v-list-item-content>
                  <v-list-item-title>{{ data.fullname }}</v-list-item-title>
                  <v-list-item-subtitle>{{ data.email }}</v-list-item-subtitle>
                </v-list-item-content>

                <v-list-item-action>
                  <v-btn color="red" @click="removeMember(data.id)">Remove</v-btn>
                </v-list-item-action>
              </v-list-item>
            </v-list>
          </v-card-text>
        </v-card>
    </v-col>

    
    <v-dialog v-model="add_member_dialog" persistent max-width="540" v-if="selectedGroup !== 0" >
      <v-card>
        <v-card-title>{{ (groups[selectedGroup]['name'] || '') }}にメンバーを追加する</v-card-title>
        <v-card-text>
          <v-autocomplete outlined :items="possibleUsers" v-model="selectedUser" item-text="fullname" item-value="ik" color="green" item-color="green"/>
        </v-card-text>
        <v-card-actions>
          <v-btn color="red" @click="add_member_dialog = false">キャンセル</v-btn>
          <v-spacer />
          <v-btn color="green" @click="addNewMember">提出</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="add_group_dialog" persistent max-width="540" v-if="selectedCourse !== 0" >
      <v-card>
        <v-card-title>{{ courses[selectedCourse]['fullname'] }}コースにグループを追加する</v-card-title>
        <v-card-text>
          <v-text-field v-model="groupFD.name" outlined label="グループ名" color="green" />

          <v-text-field v-model="groupFD.desc" outlined label="グループ概要" color="green" />
        </v-card-text>
        <v-card-actions>
          <v-btn color="red" @click="add_group_dialog = false">キャンセル</v-btn>
          <v-spacer />
          <v-btn color="green" @click="addGroup">提出</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import Swal from 'sweetalert2';

export default {
  name: 'IndexPage',

  data: () => ({
    token: null,
    loginData: {
      uname: '',
      pwd: ''
    },
      
    show: false,

    reqBody: {
        wstoken: "",
        wsfunction: "",
        moodlewsrestformat: "json"
    },

    courses: [],
    selectedCourse: 0,

    groups: [],
    selectedGroup: 0,

    members: [],
    members_ids: [],

    add_member_dialog: false,

    enrolledUsers: [],
    enrolledUsers_ids: [],
    possibleUsers: [],
    selectedUser: 0,

    add_group_dialog: false,

    groupFD: {
      name: '',
      desc: '',
    }
  }),

  watch: {
    selectedCourse: function (n) {
      if(this.courses.length > 0) {
        this.getGroups(this.courses[n]['id']);
        this.getAllUsers(this.courses[n]['id']);
      } 
    },
    selectedGroup: function(n) {
      if(this.groups.length > 0) {
        this.getMembers(this.groups[n]['id']);
      }
    }
  },

  methods: {


    revokeAccess() {
      localStorage.removeItem('ottx');
      this.token = "";
      this.members = [];
      this.groups = [];
      this.possibleUsers = [];
      this.enrolledUsers = [];
      this.courses = [];
      this.selectedUser = 0;
      this.selectedCourse = 0;
      this.selectedGroup = 0;
    },

    async getMoodleToken () {
      await fetch(`http://52.194.241.74/login/token.php?username=${this.loginData.uname}&password=${this.loginData.pwd}&service=otis_ext_lei_test`)
      .then(response => response.json())
      .then(data => {
        if(!data.token)
        {
          Swal.fire('Error', data.error, 'error')
        }
        else
        {
          Swal.fire('Success', `token ${data.token}`, 'success');
          this.token = data.token;
          localStorage.setItem("ottx", data.token);
          this.getCourses();
        }
      });
    },

    generateFormBody(data) {
      let fb = [];
      for (let p in data) {
        fb.push(encodeURIComponent(p) + "=" + encodeURIComponent(data[p]));
      }
      return fb.join("&");
    },


    reInit() {

      this.reqBody = [];
      this.reqBody = {
        wstoken: this.token,
        wsfunction: "",
        moodlewsrestformat: "json"
      }
    },


    /**
     * Get All Courses
     * 
     * function: core_course_get_courses
     */
    async getCourses() {
      this.reInit()
      this.reqBody.wsfunction = "core_course_get_courses";

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        if(data) {
          for(const key in data) {
            data[key]['apifullname'] = `[${data[key]['idnumber'] ?? " " }] ${data[key]['fullname']}`
            data[key]['ik'] = key;
            this.courses.push(data[key]);
          }
        }
      });
      
    },


    /**
     * Get Groups based in course
     * 
     * function: core_group_get_course_groups
     */
    async getGroups(courseid) {
      this.reInit()
      this.reqBody.wsfunction = "core_group_get_course_groups";
      this.reqBody["courseid"] = courseid;
      this.groups = [];
      this.selectedGroup = 0;

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        for(const key in data) {
            data[key]['ik'] = key;
            this.groups.push(data[key]);
        }
      });

    },

    /**
     * Get list of members in group
     * 
     * function: core_group_get_group_members
     */
    async getMembers(groupID) {
      this.reInit()
      this.reqBody.wsfunction = "core_group_get_group_members";
      this.reqBody["groupids[0]"] = groupID;

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        if(data) {
          this.members = [];
          this.members_ids = [];
          this.getUserDetails(data[0]['userids'], (userList) => {
            for(const key in userList) {
              userList[key]['ik'] = key;
              this.members.push(userList[key]);
              this.members_ids.push(userList[key]['id']);
            }
          });
        }
      });
    },

    /**
     * Get User Details
     * 
     * function: core_user_get_users_by_field
     */
    async getUserDetails(uids, callback) {
      if(uids.length === 0) { Swal.fire("Notice", "No Members", "info"); return; }

      this.reInit()
      this.reqBody.wsfunction = "core_user_get_users_by_field";
      this.reqBody["field"] = "id";

      for(const k in uids) {
        this.reqBody[`values[${k}]`] = uids[k];
      }

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        if(data) {
          callback(data);
        }
      });
    },



      /**
       * Get all enrollred users belong to course
       * 
       * function: core_enrol_get_enrolled_users
       */
      async getAllUsers(courseid) {
        this.reInit()
        this.reqBody.wsfunction = "core_enrol_get_enrolled_users";
        this.reqBody["courseid"] = courseid;
        this.enrolledUsers = [];
        this.selectedUser = 0;
        this.possibleUsers = [];

        await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
          },
          body: this.generateFormBody(this.reqBody)
        }).then(response => response.json()).then(data => {
          for(const key in data) {
              data[key]['ik'] = key;
              this.enrolledUsers.push(data[key]);
              this.enrolledUsers_ids.push(data[key]['id']);
          }
        });
      },

      openAddMemberDialog() {
        this.add_member_dialog = true;
        this.possibleUsers = [];
        let p_data_users = this.enrolledUsers_ids.filter((uid) => {
          return !this.members_ids.includes(uid);
        })

        this.getUserDetails(p_data_users, (userlist) => {
          if(userlist) {
            for(const key in userlist) {
              userlist[key]['ik'] = key;
              this.possibleUsers.push(userlist[key]);
            }
          }
        })
      },

    /**
     * Add new Member to the group
     * 
     * function: core_group_add_group_members
     */
    async addNewMember() {

      this.reInit()
      this.reqBody.wsfunction = "core_group_add_group_members";
      this.reqBody["members[0][groupid]"] = this.groups[this.selectedGroup]['id'];
      this.reqBody["members[0][userid]"] = this.possibleUsers[this.selectedUser]['id'];

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        console.log(data);
        Swal.fire("Success", `${ this.possibleUsers[this.selectedUser]["fullname"] } is added.`, "success");
        this.getMembers(this.groups[this.selectedGroup]['id']);
        this.add_member_dialog = false;
      });
    },

    /**
     * Remove member to the group
     * 
     * function: core_group_delete_group_members
     */
    async removeMember(uid) {

      this.reInit()
      this.reqBody.wsfunction = "core_group_delete_group_members";
      this.reqBody["members[0][groupid]"] = this.groups[this.selectedGroup]['id'];
      this.reqBody["members[0][userid]"] = uid;

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        this.getMembers(this.groups[this.selectedGroup]['id']);
      });
    },


    /**
     * Add Group
     * 
     * function: core_group_create_groups
     */
    async addGroup() {
      this.reInit()
      this.reqBody.wsfunction = "core_group_create_groups";
      this.reqBody["groups[0][name]"] = this.groupFD.name;
      this.reqBody["groups[0][description]"] = this.groupFD.desc;
      this.reqBody["groups[0][courseid]"] = this.courses[this.selectedCourse]['id'];

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        console.log(data);
        Swal.fire("Success", `${ this.groupFD.name } is added.`, "success");
        
        this.getGroups(this.courses[this.selectedCourse]['id']);
        this.getAllUsers(this.courses[this.selectedCourse]['id']);
        this.add_group_dialog = false;
      });
    },

    /**
     * Remove group
     * 
     * function: core_group_delete_groups
     */
    async removeGroup() {
      this.reInit()
      this.reqBody.wsfunction = "core_group_delete_groups";
      this.reqBody["groupids[0]"] = this.groups[this.selectedGroup]['id'];

      await fetch(`http://52.194.241.74/webservice/rest/server.php`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8'
        },
        body: this.generateFormBody(this.reqBody)
      }).then(response => response.json()).then(data => {
        console.log(data);
        this.getGroups(this.courses[this.selectedCourse]['id']);
        this.getAllUsers(this.courses[this.selectedCourse]['id']);
      });
    },
  },

  mounted() {
    this.token = localStorage.getItem('ottx');
    this.reInit();
    if(this.token !== "") {
      this.getCourses();
    }
  }
}

</script>
