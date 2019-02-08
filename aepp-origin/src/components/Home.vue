<template>
	<div class="home">
		<div class="header">
			<span v-bind:style="{ 'color': '#F03C6E' }">Æternity</span> To-do list
		</div>

		<div v-bind:class="{ waitingCall: waitingCall }"></div>

		<div class="board" v-if="this.account.pub">
			<h2 class="py-2 mt-8">
				My tasks <br/>
				Total: {{ todos.length }}
			</h2>
			<div v-bind:class="{ waitingCallOverlay: waitingCall }" class="my-tasks" v-if="todos.length > 0">
				<div class="my-tasks-inner p-4 rounded-sm shadow">
					<div class="-mx-2 mt-4 mb-4">
						<div class="to-do-content" v-bind:key="index" v-for="(todo, index) in todos" :todo.sync="todo" v-bind:class="{ completedTask: todo.done }">
							<div class='to-do-header'>
								{{ todo.title }}
							</div>
							<div class='to-do-status completed' v-show="todo.done" disabled>
								&#10004; Completed
							</div>
							<div class='to-do-status pending' v-show="!todo.done">
								Pending
							</div>
							<div class="mark-as-complete" v-show="!todo.done">
								<button class="mark-button" @click="completeTask(index)">Mark as complete</button>
							</div>
						</div>
					</div>
				</div>
			</div>
			<div v-bind:class="{ waitingCallOverlay: waitingCall }" class="create-task-holder">
				<div class="create-task-input-holder">
					<input v-model="createTaskInput" class="w-full p-2" type="text" id="create-task-input" placeholder="Task name">
				</div>
				<div id="create-task">
					<button class="ae-button" @click="createTask">Create task</button>
				</div>
			</div>
			<div class="mt-2 mb-2 call-result" v-if="callRes && !callError">
				<label class="text-xs block mb-1">Last Call Result</label>
				<div class="w-full text-white bg-black text-xs mb-4 p-4 font-mono" v-html="callRes">
				</div>
			</div>
		</div>

		<div class="error-message-holder">
      <span v-if="this.clientError" class="text-lg text-red">
          {{ this.clientError }}
        </span>
		</div>
	</div>
</template>

<script>
  //  is a webpack alias present in webpack.config.js
  import Aepp from '@aeternity/aepp-sdk/es/ae/aepp'
  import settingsData from '../settings.js'

  export default {
    name: 'Home',
    components: {},
    data() {
      return {
        showModal: false,
        showBoard: false,
        privateKeyInput: '',
        account: { priv: null, pub: null },
        balance: 0,
        balanceInterval: null,
        client: false,
        contractAddress: settingsData.deployedContractAddress,
        callOpts: {
          deposit: 1,
          gasPrice: 1,
          amount: 1,
          callData: ''
        },
        clientError: false,
        callRes: '',
        callError: '',
        deployError: '',
        compileError: '',
        callStaticRes: '',
        callStaticError: '',
        waitingCall: false,
        sophiaType: 'int',
        sophiaTypes: [
          'uint',
          'int',
          'address',
          'bool',
          'string',
          'list',
          'tuple',
          'record',
          'map',
          'option(\'a)',
          'state',
          'transactions',
          'events',
          'oracle(\'a, \'b)',
          'oracle_query(\'a, \'b)'
        ],
        todos: [],
        createTaskInput: '',
        returnedData: null
      }
    },
    computed: {},
    methods: {
      callContract(func, args, options) {
        console.log(`calling a function on a deployed contract with func: ${func}, args: ${args} and options:`, options)
        return this.client.contractCall(this.contractAddress, 'sophia-address', this.contractAddress, func, { args, options })
      },
      callStatic(func, args) {
        console.log(`calling static func ${func} with args ${args}`)
        return this.client.contractCallStatic(this.contractAddress, 'sophia-address', func, { args })
      },
      async onCallStatic(funcName, funcArgs, returnType) {
        if (funcName && funcArgs && returnType) {
          this.waitingCall = true
          try {
            const dataRes = await this.callStatic(funcName, funcArgs)
            this.callRes = await dataRes.decode(returnType)
            const dataValue = this.callRes.value
            console.log(dataValue)

            this.callRes = `Function: ${funcName} <br><br>---<br><br> Result: <br><br> ${dataValue}`
            this.callError = ''
            this.waitingCall = false

            return dataValue
          } catch (err) {
            this.callError = `${err}`
            this.waitingCall = false
          }
        } else {
          this.callStaticError = 'Please enter a Function and 1 or more Arguments.'
        }
      },
      async assignBalance(accountPub) {
        return this.client.balance(accountPub).then(balance => {
          return balance
        })
      },
      async onCallDataAndFunctionAsync(funcName, funcArgs, returnType) {
        const extraOpts = {
           'owner': this.account.pub
          // 'vmVersion': 1
          // 'nonce': 0,
          // 'ttl': 9999999
        }
        const opts = Object.assign(extraOpts, this.callOpts)

        if (funcName && funcArgs && returnType) {
          this.waitingCall = true
          try {
            const dataRes = await this.callContract(funcName, funcArgs, opts)
            this.callRes = dataRes.result

            const data = await this.client.contractDecodeData(returnType, dataRes.result.returnValue)
            this.returnedData = data
            this.callRes = `Function: ${funcName} <br><br>---<br><br> Gas Used: ${dataRes.result.gasUsed} <br><br>---<br><br> Result: <br><br> ${data.value}`
            this.callError = ''
            this.waitingCall = false
            this.createTaskInput = ''
            return data
          } catch (err) {
            this.callError = `${err}`
            this.waitingCall = false
          }
        } else {
          this.callError = 'Please enter a Function and 1 or more Arguments.'
        }
      },
      async completeTask(taskIndex) {
        const completed = await this.onCallDataAndFunctionAsync('complete_task', `(${taskIndex})`, 'bool')
        console.log(completed.value)
        this.todos[taskIndex].done = true
      },
      async createTask() {
        if (this.createTaskInput) {
          const taskName = await this.onCallDataAndFunctionAsync('add_to_do', `("${this.createTaskInput}")`, 'string')
          const task = {
            title: taskName.value,
            done: false
          }
          this.todos.push(task)
        }
      },
      async getContractTasks() {
        const taskCount = await this.onCallStatic('get_task_count', '()', 'int')
        let taskName
        let taskCompleted

        for (let i = 0; i < taskCount; i++) {
          taskName = await this.onCallStatic('get_task_by_index', `(${i})`, 'string')
          taskCompleted = await this.onCallStatic('task_is_completed', `(${i})`, 'bool')
          console.log(taskCompleted)
          const task = {
            title: taskName,
            done: !!taskCompleted
          }
          this.todos.push(task)
          console.log(this.todos)
        }
      }
    },
    created() {
      Aepp().then(ae => {
        this.client = ae
        console.log(this.client);
        ae.address()
          .then(address => {
            this.account.pub = address
            console.log(address);
            this.getContractTasks()
          })
          .catch(e => {
            this.account.pub = `Rejected: ${e}`
          })
      })
    }
  }
</script>

<style lang="css">

	.header {
		background-color: #f5f5f5;
		height: 100px;
		font-size: 30px;
		padding: 30px;
	}

	.board {
		margin: 30px;
	}

	.my-tasks {
		max-height: 600px;
		overflow-y: auto;
		overflow-x: hidden;
		border-radius: 12px;
		border: 1px solid lightgray;
		background-color: aliceblue;
		margin-top: 20px;
	}

	.my-tasks-inner {
		width: 60%;
		margin: 0 20%;
	}

	#modal-button-holder {
		text-align: center;
		margin-top: 40px;
	}

	.to-do-content {
		display: inline-block;
		margin: 0 50px 30px 0;
		width: 100%;
		border: 1px solid slategray;
		padding: 20px;
		border-radius: 12px;
		background-color: deepskyblue;
	}

	.to-do-content.completedTask {
		background-color: #72f38d;
	}

	.create-task-input-holder {
		display: inline-block;
		width: 79%;
	}

	#create-task {
		display: inline-block;
		width: 20%;
		text-align: right;
	}

	.create-task-holder {
		padding: 15px;
		border: 1px solid lightgray;
		border-radius: 12px;
		margin-top: 15px;
		background-color: aliceblue;
	}

	.to-do-header {
		background-color: #f5f5f5;
		height: 80px;
		border-radius: 10px;
		text-align: center;
		padding-top: 26px;
		font-size: 26px;
		border: 1px solid #F03C6E;
	}

	.to-do-status.completed {
		float: right;
		font-size: 20px;
		margin-top: 15px;
		color: black;
		padding: 8px 0;
		border-radius: 10px;
	}

	.to-do-status.pending {
		float: left;
		font-size: 20px;
		margin-top: 15px;
		background-color: deepskyblue;
		color: white;
		padding: 8px 0;
		border-radius: 10px;
	}

	.mark-as-complete {
		float: right;
	}

	.ae-button {
		background-color: #F03C6E;
		-moz-border-radius: 28px;
		-webkit-border-radius: 28px;
		border-radius: 28px;
		border: 1px solid #F03C6E;
		display: inline-block;
		cursor: pointer;
		color: #ffffff;
		font-size: 17px;
		padding: 16px 31px;
		text-decoration: none;
		text-shadow: 0px 1px 0px #F03e6E;
		outline: none;
	}

	.ae-button:hover {
		background-color: #F90C6E;
	}

	.ae-button:active {
		position: relative;
		top: 1px;
		outline: none;
	}

	#privateKeyInput {
		border: solid 1px lightgray;
		border-radius: 5px;
		margin: 20px 0 200px;
		outline: none;
	}

	.mark-button {
		background-color: #301B58;
		-moz-border-radius: 28px;
		-webkit-border-radius: 28px;
		border-radius: 28px;
		border: 1px solid #301B58;
		display: inline-block;
		cursor: pointer;
		color: #ffffff;
		font-size: 17px;
		padding: 10px 20px;
		margin-top: 10px;
		text-decoration: none;
		text-shadow: 0px 1px 0px #301B58;
		outline: none;
	}

	.mark-button:hover {
		background-color: black;
	}

	#create-task-input {
		border: 1px solid #F03C6E;
		border-radius: 12px;
		margin: 20px 45px 20px 0;
		outline: none;
		width: 100%;
		height: 80px;
	}

	.call-result {
		background-color: black;
		color: #72f38d;
		padding: 20px;
		border-radius: 10px;
	}

	.modal-mask {
		position: fixed;
		z-index: 9998;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background-color: rgba(0, 0, 0, .5);
		display: table;
		transition: opacity .3s ease;
	}

	.modal-wrapper {
		display: table-cell;
		vertical-align: middle;
	}

	.modal-container {
		width: 600px;
		height: 400px;
		margin: 0 auto 500px;
		padding: 20px 30px;
		background-color: #fff;
		border-radius: 12px;
		box-shadow: 0 2px 8px rgba(0, 0, 0, .33);
		transition: all .3s ease;
		font-family: Helvetica, Arial, sans-serif;
	}

	.modal-header h3 {
		margin-top: 0;
		color: #42b983;
	}

	.modal-default-button {
		float: right;
	}

	/*
	 * The following styles are auto-applied to elements with
	 * transition="modal" when their visibility is toggled
	 * by Vue.js.
	 *
	 * You can easily play with the modal transition by editing
	 * these styles.
	 */

	.modal-enter {
		opacity: 0;
	}

	.modal-leave-active {
		opacity: 0;
	}

	.modal-enter .modal-container,
	.modal-leave-active .modal-container {
		-webkit-transform: scale(1.1);
		transform: scale(1.1);
	}

	.waitingCall {
		position: fixed;
		z-index: 999;
		height: 2em;
		width: 2em;
		overflow: visible;
		margin: auto;
		top: 0;
		left: 0;
		bottom: 0;
		right: 0;
	}

	/* Transparent Overlay */
	.waitingCall:before {
		content: '';
		display: block;
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background-color: rgba(0, 0, 0, 0.3);
	}

	/* :not(:required) hides these rules from IE9 and below */
	.waitingCall:not(:required) {
		/* hide "waitingCall..." text */
		font: 0/0 a;
		color: transparent;
		text-shadow: none;
		background-color: transparent;
		border: 0;
	}

	.waitingCall:not(:required):after {
		content: '';
		display: block;
		font-size: 10px;
		width: 1em;
		height: 1em;
		margin-top: -0.5em;
		-webkit-animation: spinner 1500ms infinite linear;
		-moz-animation: spinner 1500ms infinite linear;
		-ms-animation: spinner 1500ms infinite linear;
		-o-animation: spinner 1500ms infinite linear;
		animation: spinner 1500ms infinite linear;
		border-radius: 0.5em;
		-webkit-box-shadow: rgba(0, 0, 0, 0.75) 1.5em 0 0 0, rgba(0, 0, 0, 0.75) 1.1em 1.1em 0 0, rgba(0, 0, 0, 0.75) 0 1.5em 0 0, rgba(0, 0, 0, 0.75) -1.1em 1.1em 0 0, rgba(0, 0, 0, 0.5) -1.5em 0 0 0, rgba(0, 0, 0, 0.5) -1.1em -1.1em 0 0, rgba(0, 0, 0, 0.75) 0 -1.5em 0 0, rgba(0, 0, 0, 0.75) 1.1em -1.1em 0 0;
		box-shadow: rgba(0, 0, 0, 0.75) 1.5em 0 0 0, rgba(0, 0, 0, 0.75) 1.1em 1.1em 0 0, rgba(0, 0, 0, 0.75) 0 1.5em 0 0, rgba(0, 0, 0, 0.75) -1.1em 1.1em 0 0, rgba(0, 0, 0, 0.75) -1.5em 0 0 0, rgba(0, 0, 0, 0.75) -1.1em -1.1em 0 0, rgba(0, 0, 0, 0.75) 0 -1.5em 0 0, rgba(0, 0, 0, 0.75) 1.1em -1.1em 0 0;
	}

	/* Animation */

	@-webkit-keyframes spinner {
		0% {
			-webkit-transform: rotate(0deg);
			-moz-transform: rotate(0deg);
			-ms-transform: rotate(0deg);
			-o-transform: rotate(0deg);
			transform: rotate(0deg);
		}
		100% {
			-webkit-transform: rotate(360deg);
			-moz-transform: rotate(360deg);
			-ms-transform: rotate(360deg);
			-o-transform: rotate(360deg);
			transform: rotate(360deg);
		}
	}

	@-moz-keyframes spinner {
		0% {
			-webkit-transform: rotate(0deg);
			-moz-transform: rotate(0deg);
			-ms-transform: rotate(0deg);
			-o-transform: rotate(0deg);
			transform: rotate(0deg);
		}
		100% {
			-webkit-transform: rotate(360deg);
			-moz-transform: rotate(360deg);
			-ms-transform: rotate(360deg);
			-o-transform: rotate(360deg);
			transform: rotate(360deg);
		}
	}

	@-o-keyframes spinner {
		0% {
			-webkit-transform: rotate(0deg);
			-moz-transform: rotate(0deg);
			-ms-transform: rotate(0deg);
			-o-transform: rotate(0deg);
			transform: rotate(0deg);
		}
		100% {
			-webkit-transform: rotate(360deg);
			-moz-transform: rotate(360deg);
			-ms-transform: rotate(360deg);
			-o-transform: rotate(360deg);
			transform: rotate(360deg);
		}
	}

	@keyframes spinner {
		0% {
			-webkit-transform: rotate(0deg);
			-moz-transform: rotate(0deg);
			-ms-transform: rotate(0deg);
			-o-transform: rotate(0deg);
			transform: rotate(0deg);
		}
		100% {
			-webkit-transform: rotate(360deg);
			-moz-transform: rotate(360deg);
			-ms-transform: rotate(360deg);
			-o-transform: rotate(360deg);
			transform: rotate(360deg);
		}
	}

</style>
