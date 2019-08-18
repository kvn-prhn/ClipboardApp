<template>
	<div>
		<div class="content-row">
			<filters @filterApplied="updateEvents()"></filters>
			<div class="content-col event-container">
				<event-list :events="events"></event-list>
				<paginate
					:page-count="10"
					:click-handler="paginateHandler"
					:prev-text="'Prev'"
					:next-text="'Next'"
					:container-class="'pagination'"
					:page-class="'page-item'">
				</paginate>
			</div>
		</div>
		<no-ssr>
			<notifications group="default"/>
		</no-ssr>
	</div>	
</template>

<script lang='ts'>
	import axios from 'axios';
	import Filters from '~/components/Filters.vue';
	import EventList from '~/components/EventList.vue';
	
	import { dummyData } from '~/store/dummyData.js';
	
	// function getClient(url: string): Service<any> {
	// 	const app = feathers();
	// 	const restClient = rest('http://' + url);
	// 	app.configure(restClient.axios(axios));
	// 	return app.service('events');
	// }
	function getEventURL(in2itApiUrl) {
		const eventURL = process.server ? 'event_service:5000' : in2itApiUrl;
		return `http://${eventURL}/events`;
	}

	export default {
		data() {
			return {
				events: [],
				pageSize: 10
			};
		},
		asyncData ({ app, params }) {
            //Ensure get request goes to an endpoint that returns an array or json object
            //If a regular HTML page is returned, the v-for in the view above will try to
            //render each character in the HTML page string as a separate event and nuxt
			//will run out of memory
            if (process.env.DUMMY_DATA) {
                return { events: dummyData };
			}

			return axios.get(getEventURL(app.$env.IN2IT_API_URL), {
				params: {
					limit: 10
				}
			})
			.then(res => {
				return { events: res.data };
			});
		},
		methods: {
			updateEvents: function() {
				// Manually set the time to 11:59 PM for now because we don't have a time picker yet
				this.$store.searchFilter.endDate.setHours(23, 59, 59);

				return axios.get(getEventURL(this.$env.IN2IT_API_URL), {
					params: {
						startTime: this.$store.searchFilter.startDate, 
						endTime: this.$store.searchFilter.endDate,
						miles: this.$store.searchFilter.searchRadius,
						address: this.$store.searchFilter.addressOrZip,
						organization: this.$store.searchFilter.organization,
						neighborhood: this.$store.searchFilter.neighborhood,
						limit: this.pageSize,
						offset: (this.$store.searchFilter.pageNum - 1) * this.pageSize
					}
				})
				.then((res) => {
					this.events = res.data;

					this.$notify({
						group: 'default',
						title: 'Filters applied',
						type: 'success'
					});
				})
				.catch((res) => {
					this.$notify({
						group: 'default',
						title: 'Error applying filters',
						type: 'error'
					});
				});
			},
			paginateHandler: function(pageNum){
				this.$store.searchFilter.pageNum = pageNum;
				this.updateEvents();
			}
		},
		components: {
			Filters,
			EventList
		}
	};
</script>