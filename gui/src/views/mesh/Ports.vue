<script setup>
import { ref,onActivated,watch, computed } from "vue";
import { useRouter } from 'vue-router'
import ZtmService from '@/service/ZtmService';
import { useStore } from 'vuex';
import { useConfirm } from "primevue/useconfirm";

const store = useStore();
const router = useRouter();
const ztmService = new ZtmService();
const confirm = useConfirm();
const loading = ref(false);
const loader = ref(false);
const status = ref({});
const endpointMap = ref({});
const ports = ref([])

const props = defineProps(['ep','embed']);
const meshes = computed(() => {
	return store.getters['account/meshes']
});

const isChat = computed(() => store.getters['account/isChat']);
const selectedMesh = computed(() => {
	return store.getters["account/selectedMesh"]
});
onActivated(()=>{
	getPorts();
	getEndpoints();
})
const deletePort = (port) => {
	ztmService.deletePort({
		mesh:selectedMesh.value?.name,
		ep:selectedMesh.value?.agent?.id,
		proto: port.protocol,
		ip: port?.listen?.ip,
		port: port?.listen?.port
	},()=>{
		getPorts()
	});
}

const getEndpoints = () => {
	ztmService.getEndpoints(selectedMesh.value?.name)
		.then(res => {
			res.forEach((ep) => {
				endpointMap.value[ep.id] = ep;
			})
		})
		.catch(err => console.log('Request Failed', err)); 
}
const error = ref();
const getPorts = () => {
	const _ep = props.embed?props.ep:selectedMesh.value?.agent?.id;
	if(!!_ep){
		loading.value = true;
		loader.value = true;
		setTimeout(()=>{
			ztmService.getPorts({
				mesh:selectedMesh.value?.name,
				ep:_ep
			})
				.then(res => {
					console.log("ports:")
					console.log(res)
					loading.value = false;
					setTimeout(() => {
						loader.value = false;
					},1400)
					error.value = null;
					ports.value = res || [];
				})
				.catch(err => {
					error.value = err;
					loading.value = false;
					loader.value = false;
				}); 
		},600);
	}
}
watch(()=>selectedMesh,()=>{
	if(selectedMesh.value){
		getPorts();
		getEndpoints();
	}
},{
	deep:true,
	immediate:true
})
const portsFilter = computed(() => {
	console.log(typeof(ports.value))
	console.log(ports.value)
	return !!ports.value?ports.value.filter((port)=>{
		return (typing.value == '' || typing.value == port.target.service|| typing.value == port.listen.port );
	}):[]
});

const typing = ref('');

const emptyMsg = computed(()=>{
	const _ep = props.embed?props.ep:selectedMesh.value?.agent?.id;
	if(!!_ep){
		return 'No port.'
	} else {
		return `First, join a ${isChat.value?'Channel':'Mesh'}.`
	}
});
</script>

<template>
	<div class="flex flex-row min-h-screen" :class="{'embed-ep-header':props.embed}" >
		<div class="relative w-full h-full" >
			<AppHeader :main="!isChat" v-if="!props.embed">
					<template #center>
						<b>Local Ports ({{portsFilter.length}})</b>
					</template>
			
					<template #end> 
						<Button icon="pi pi-refresh" text @click="getPorts"  :loading="loader"/>
					</template>
			</AppHeader>
			<Card class="nopd" v-if="!error">
				<template #content>
					<InputGroup class="search-bar" >
						<Button :disabled="!typing" icon="pi pi-search" :label="selectedMesh?.name" />
						<Textarea @keyup="watchEnter" v-model="typing" :autoResize="true" class="drak-input bg-gray-900 text-white flex-1" placeholder="Type service | port" rows="1" cols="30" />
					</InputGroup>
				</template>
			</Card>
			<Loading v-if="loading"/>
			<ScrollPanel class="w-full absolute" style="bottom: 0;" :style="{'top':props.embed?'35px':'75px'}" v-else-if="portsFilter && portsFilter.length >0">
			<div  class="text-center">
				<div class="mt-1 grid text-left px-3">
						<div class="col-12" :class="props.embed?'md:col-6 lg:col-4':'md:col-6 lg:col-3'" v-for="(port,hid) in portsFilter" :key="hid">
							 <div class="surface-card shadow-2 p-3 border-round">
									 <div class="flex justify-content-between mb-1">
											 <div>
													<span class="block text-tip font-medium mb-3">
														 {{port.listen.ip}} | {{port.protocol}}
													</span>
													
													<div class="text-900 font-medium text-xl">
														<Status :run="port.open" :error="port.error" :text="port.listen.port"/>
													</div>
											 </div>
											 <div class="flex">
												 <div v-tooltip="'Delete Port'"  @click="deletePort(port)" class="pointer flex align-items-center justify-content-center p-button-secondary border-round mr-2" style="width: 2.5rem; height: 2.5rem">
														 <i class="pi pi-trash text-tip text-xl"></i>
												 </div>
											 </div>
									 </div>
										<Fieldset legend="Target" :toggleable="true" >
											<div class="mb-2" v-if="port?.mesh">
												<Chip class="pl-0 pr-3 mr-2">
														<span class="bg-primary border-circle w-2rem h-2rem flex align-items-center justify-content-center">
															<i class="pi pi-globe"/>
														</span>
														<span class="font-medium ml-2">
															Mesh: {{decodeURI(port.mesh)}}
														</span>
												</Chip>
											</div>
											<div class="mb-2">
												<Chip class="pl-0 pr-3 mr-2">
														<span class="bg-primary border-circle w-2rem h-2rem flex align-items-center justify-content-center">
															<i class="pi pi-server"/>
														</span>
														<span class="font-medium ml-2">
															Service: {{port.target?.service}}
														</span>
												</Chip>
											</div>
											<div class="mb-2" v-if="port.target?.endpoint">
												<Chip class="pl-0 pr-3 mr-2">
														<span style="min-width: 28px;" class="bg-primary border-circle w-2rem h-2rem flex align-items-center justify-content-center">
															<i class="pi pi-chart-scatter"/>
														</span>
														<span class="font-medium ml-2">
															EP: {{endpointMap[port.target?.endpoint]?.name||'Unnamed EP' }}
														</span>
												</Chip>
											</div>
										</Fieldset>
							 </div>
					 </div>
				</div>
			</div>
			</ScrollPanel>
			<Empty v-else :title="emptyMsg" :error="error"/>
		</div>
	</div>
</template>

<style scoped lang="scss">
:deep(.p-dataview-content) {
  background-color: transparent !important;
}
:deep(.p-tabview-nav),
:deep(.p-tabview-panels),
:deep(.p-tabview-nav-link){
	background: transparent !important;
}
:deep(.p-tabview-panels){
	padding: 0;
}
</style>