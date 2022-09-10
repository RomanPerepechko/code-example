<template>
  <div class='filter-app'>
    <div class="filter-app__main">
        <div class="filter-app__filter-button button" @click="flags.isOpenedFilter = true">   
            <svg class="filter-app__toggle-svg">
                <use xlink:href="#ic-holey-filter"></use>
            </svg>
            <span class="filter-app__filter-button-title">
                Фильтр   
            </span>
      </div>
      <div class="filter-app__filters"
           :class="{'_active': flags.isOpenedFilter}">
        <div class="filter-app__filter-list"
             ref="filterList">
          <h2 class="filter-app__filter-title">Фильтр</h2>

          <FilterList :filters="filters"
                      :facets="facets"
                      :values="values"
                      :search="search"
                      :isFull="flags.isFullMode"
          />
        </div>

        <div class="filter-app__controllers _mobile-only">
          <div class="filter-app__controllers-toggle">
            <FilterFullToggle :flags="flags"
                              @toggle="flags.isFullMode = !flags.isFullMode"
            />
          </div>

          <FilterSubmit :group="group"
                        :count="service.count"
                        :disabled="this.flags.isReloading || !service.count"
                        :flags="flags"
                        :types="types"
                        :type="type"
                        class="filter-app__submit"
                        @filter-submit="onSubmitFilter"
          />

          <div class="filter-app__reset"
               @click="onFilterReset">
            <svg class="filter-app__reset-icon">
              <use xlink:href="#ic-cross"></use>
            </svg>
            <div class="filter-app__reset-text">Сбросить фильтр</div>
          </div>
        </div>
      </div>

      <div class="filter-app__controllers _not-mobile">
        <div class="filter-app__controllers-toggle">
          <FilterFullToggle :flags="flags"
                            @toggle="flags.isFullMode = !flags.isFullMode"
          />
        </div>

        <FilterSubmit :group="group"
                      :count="service.count"
                      :disabled="this.flags.isReloading || !service.count"
                      :flags="flags"
                      :types="types"
                      :type="type"
                      class="filter-app__submit"
                      @filter-submit="onSubmitFilter"
        />

        <div class="filter-app__reset"
             @click="onFilterReset">
          <svg class="filter-app__reset-icon">
            <use xlink:href="#ic-cross"></use>
          </svg>
          <div class="filter-app__reset-text">Сбросить фильтр</div>
        </div>
      </div>

      <control-panel :controls="controls"
                     :group="group"
                     :count="service.count"
                     :flags="flags"
                     :types="types"
                     :type="type"
      />
    </div>

    <div class="filter-app__content">
      <div class="filter-app__list" :class="{_hide: flags.isReloading}">
        <Type v-if="flags.isOpenedType"
              :controls="controls"
              :type="type"
              :count="service.count"
              :mixed-objects="flats"
              :origin-objects="initialFlats"
        />

        <TypeList v-else-if="group && types && types.length"
                  :mixed-objects="types"
                  :origin-objects="initialTypes"
                  :isCells="flags.isCells"
        />

        <FlatList v-else
                  :mixed-objects="flats"
                  :origin-objects="initialFlats"
                  :isCells="flags.isCells"
        />
      </div>

        <drag-drop :flags="flags"
                   :group="group"
                   :fav-counter="favoriteCounter"
                   @dragenter="flags.isDragOver = true"
                   @dragleave="flags.isDragOver = false"
                   :miniature="miniature"
        />

    </div>
  </div>
</template>


<script>
    import {Utils} from '../../common/Classes/Utils';
    import {MindboxUtils} from '../../mindbox/MindboxUtils';

    import FilterList from './blocks/FilterList.vue';
    import FilterSubmit from './filter/FilterSubmit.vue'
    import FilterFullToggle from './filter/FilterFullToggle.vue'
    import ControlPanel from './blocks/ControlPanel.vue';
    import FlatList from './blocks/FlatList.vue';
    import TypeList from './blocks/TypeList.vue';
    import Type from './blocks/Type.vue';
    import DragDrop from './blocks/DragDrop.vue';
    import {eventPush} from '../../common/mindbox/eventPush';
    import {sendFlats} from '../../common/mindbox/sendFlats';

    const upsaleCardTypes = ['priceIncrease', 'penthouse', 'phaseIsFinished', 'projectFinish', 'nextProject', 'parking', 'rareText', 'roomIncrease', 'callback', 'commercial'];
    const title = document.querySelector('.js-flats-title');

    export default {
        name: 'FilterApp',

        props: {
            filters: Array,
            routes: {
                baseUrl: String,
                facetsUrl: String,
                flatFilterUrl: String,
                typeFilterUrl: String,
            },
            defaultQueryState: {
                type: String,
                default() {
                    return null;
                }
            },
            miniature: {
                type: Boolean,
                default: false,
            }
        },

        components: {
            DragDrop,
            FilterList,
            FilterSubmit,
            FilterFullToggle,
            ControlPanel,
            FlatList,
            TypeList,
            Type
        },

        data: function () {
            return {
                values: {},
                controls: {
                    ordering: '',
                    type: '',
                },
                group: '',
                service: {
                    count: 0,
                    upsalesCount: 0,
                    lastStaticUpsaleIndex: 0,
                    lastCustomerUpsaleIndex: 0,
                    limit: 8,
                    offset: 0,
                },
                upsalesInfo: {},
                search: '',
                flags: {
                    allowScrollLoad: false,
                    isFullMode: false,
                    isOpenedType: false,
                    isCells: false,
                    isReloading: false,
                    isDragging: false,
                    isDragOver: false,
                    isOpenedFilter: true,
                },
                filterTypes: this.prebuildFilterData(this.filters, 'type'),
                flats: [],
                initialFlats: [],
                types: [],
                initialTypes: [],
                slicedObjects: [],
                type: {},
                facets: {},
                isUserAction: false,
                favoriteCounter: null,
                favApi: '/api/favorites/count/',
                additionalFilters:{},
                sentCriteo: false,
                /** используется для хранения той value, которую только что поменяли в фильтре */
                updatedValue: {},
            }
        },

        created: function () {
            const query = this.defaultQueryState ? this.defaultQueryState : window.location.search;

            this.prepareState(this.queryParse(query));
            if (this.values.project && this.values.project.length) {
                this.updatedValue.project = [...this.values.project];
            }

            const projectPointOfContactSlug = this.values && this.values.project && this.values.project.length === 1 ?
                this.values.project[0] :
                null;

            MindboxUtils.changeEndPoint({shortSlug: projectPointOfContactSlug})
                .then(() => {
                    this.pushMindboxEvent('Page.Filter');
                });
        },

        mounted: function () {
            this.getFavCount();

            if (this.controls.type) {
                this.onLoadType();
            } else {
                this.getObjects();
            }

            setTimeout(() => this.flags.isOpenedFilter = false, 1);

            this.$root.$on('update-values', (val, mindboxOperation) => {
                /** используется для отправки квартир в criteo */
                if (val.project) {
                    const newVal = val.project.find(el => !this.values.project.includes(el));
                    this.$set(this.updatedValue, 'project', newVal);
                }

                this.values = {...this.values, ...val};

                if (this.values.project && this.values.project.includes(null) && this.values.building && this.values.building.length) {
                    this.values.building = []
                }

                const projectPointOfContactSlug = this.values && this.values.project && this.values.project.length === 1 ?
                    this.values.project[0] :
                    null;

                MindboxUtils.changeEndPoint({shortSlug: projectPointOfContactSlug})
                    .then(() => {
                        mindboxOperation && this.pushMindboxEvent(mindboxOperation);
                    });

                this.resetAlterRoom();
                this.onUserAction();
                this.$root.$emit('filter-submit');
                this.sentCriteo = false;

                this.fixTitle();
            });

            this.$root.$on('update-controls', (val, mindboxOperation) => {
                this.controls = {...this.controls, ...val};

                mindboxOperation && this.pushMindboxEvent(mindboxOperation);

                this.resetAlterRoom();
                this.updateObjects();
                this.mobileScrollTop();
                this.onUserAction();
            });

            this.$root.$on('update-group', (value) => {
                this.group = value;

                this.resetAlterRoom();
                this.updateObjects();
                this.mobileScrollTop();
                this.onUserAction();
            });

            this.$root.$on('update-search', value => {
                this.search = value;

                this.resetAlterRoom();
                this.onFilterReset(false);
            });

            this.$root.$on('load-type', (typePk) => {
                this.controls.type = typePk;
                this.onLoadType();
                this.onUserAction();
            });

            this.$root.$on('back-from-type', () => {
                this.flags.isReloading = true;

                /* animate hardcode */
                setTimeout(() => {
                    this.initialFlats = [];
                    this.type = {};
                    this.controls.type = '';
                    this.group = true;
                    this.flags.isOpenedType = false;

                    this.updateObjects()
                        .then(() => {
                            this.flags.isReloading = false;
                            this.onUserAction();
                            this.mobileScrollTop();
                        });
                }, 700);
            });

            this.$root.$on('update-view', (val) => {
                this.flags.isCells = val;
                this.flags.isReloading = true;
                this.onUserAction();

                setTimeout(() => {
                    this.flags.isReloading = false;
                }, 700);
            });

            this.$root.$on('drag-start', () => {
                this.flags.isDragging = true;
            });

            this.$root.$on('drag-end', () => {
                this.flags.isDragging = false;
                this.flags.isDragOver = false;
            });

            this.$root.$on('add-fav', (object) => {
                Utils.toggleFavorite(object).then(() => {
                    this.getFavCount();
                });
            });

            this.$root.$on('filter-submit', () => {
                if (!this.defaultQueryState) {
                    window.history.replaceState(null, null, this.baseUrlString);
                }

                this.service.offset = 0;
                this.onUserAction();

                this.getObjects()
                    .then(() => {
                        if (!this.group) {
                            const flats = this.initialFlats.slice(0, 5);

                            MindboxUtils.flatFilterChange(flats);
                        }
                    });

                this.$emit('filter-update');
            });

            window.addEventListener('scroll', this.scrollLoading);

            this.fixTitle();
        },

        computed: {
            baseUrlString() {
                let url = this.routes.baseUrl + '?';
                let param = this.queryStringify(this.values);

                param = param.split('&').filter(item => item.indexOf('null') === -1).join('&');

                for (let option in this.controls) {
                    if (this.controls[option]) {
                        url += `${option}=${this.controls[option]}&`;
                    }
                }

                if (param) {
                    url += param + '&';
                }

                if (this.group && !this.flags.isOpenedType) {
                    url += `group=${+this.group}&`;
                }

                return url.slice(0, -1);
            },

            filterUrl() {
                return this.group ? this.routes.typeFilterUrl : this.routes.flatFilterUrl;
            },
        },

        watch: {
            'flags.isFullMode': function (val) {
                const bg = document.getElementById('flat-list-bg');

                if (val) {
                    bg.classList.add('_full');

                    if (this.$mq === 'mobile' && this.$refs.filterList) {
                        setTimeout(() => Utils.scrollElement(this.$refs.filterList, 170), 200);
                    }
                } else {
                    bg.classList.remove('_full');
                }
            }
        },

        methods: {
            async getFavCount() {
                return fetch(this.favApi, {method: 'GET'})
                    .then(res => res.json())
                    .then(({count}) => {
                        this.changeFavCount(count);
                        return count;
                    });
            },

            changeFavCount(count){
                this.favoriteCounter = count;
            },

            prebuildFilterData: function (filters, mode = 'value') {
                const result = {};

                filters.forEach((filter) => {
                    if (mode === 'value') {
                        if (filter.name === 'project' || filter.name === 'completion' || filter.name === 'category' || filter.name === 'building' || filter.name === 'renovation') {
                            result[filter.name] = [null];
                        }
                        // else if (filter.name === 'renovation') {
                        //     result[filter.name] = ['null'];
                        // }
                        else {
                            result[filter.name] = [];
                        }
                    }
                    if (mode === 'type') {
                        result[filter.name] = filter.type;
                    }
                });

                return result;
            },

            prepareState: function (params) {
                let values = this.prebuildFilterData(this.filters, 'value');

                if (params) {
                    for (let name in params) {
                        if (this.controls.hasOwnProperty(name)) {
                            this.controls[name] = params[name].join(',');
                        } else if (name === 'group') {
                            this.group = +params[name];
                        } else if (values.hasOwnProperty(name)) {
                            values[name] = params[name];
                        }else {
                            this.additionalFilters[name] = params[name];
                        }
                    }
                }
                this.values = Object.assign({}, this.values, values);
            },

            loadFacets: function () {
                let url = `${this.routes.facetsUrl}?`;
                let param = this.queryStringify(this.values);

                param = param.split('&').filter(item => item.indexOf('null') === -1).join('&');

                if (this.search) {
                    url += `article=${this.search}`;
                } else {
                    for (let option in this.controls) {
                        if (this.controls[option]) {
                            url += `${option}=${this.controls[option]}&`;
                        }
                    }

                    if (param) {
                        url += param;
                    }

                    if (this.group && !this.flags.isOpenedType) {
                        url += '&group=1&'
                    }
                }

                return fetch(url, {
                    credentials: 'same-origin'
                })
                    .then(response => response.json())
                    .then(data => {
                        this.service.count = data.count;
                        this.facets = data.facets;
                    });
            },

            scrollLoading: async function () {
                let fetchData;
                const isNotReceivedAllObjects = this.service.count > this.service.offset;

                if (this.search) {
                    // load flat by search field
                    const fetchData = await this.loadSearchFlat();

                    this.initialFlats = this.initialFlats.concat(fetchData.results);
                } else if (this.flags.allowScrollLoad && window.pageYOffset + window.innerHeight > document.body.clientHeight - 2500) {
                    this.flags.allowScrollLoad = false;

                    if (isNotReceivedAllObjects) {
                        if (this.group && !this.flags.isOpenedType) {
                            // load Types
                            fetchData = await this.loadTypes();
                        } else {
                            // load flats
                            fetchData = await this.loadFlats();
                        }
                    }

                    const fetchDataObjects = fetchData && fetchData.results || [];
                    const objects = [...this.slicedObjects, ...fetchDataObjects];

                    const {
                        initialObjects,
                        mixedObjects,
                        slicedObjects,
                    } = this.mixUpsales(objects);

                    this.loadObjectsFromData(mixedObjects, initialObjects, true);

                    this.service.offset += fetchDataObjects.length;
                    this.slicedObjects = slicedObjects;

                    if (isNotReceivedAllObjects || this.slicedObjects.length) {
                        this.flags.allowScrollLoad = true;
                    }
                }

                this.onUserAction();
            },

            getObjects: async function () {
                this.flags.isReloading = true;
                let start = Date.now();
                let stop;
                let fetchData;
                await this.loadFacets();

                this.resetUpsales();

                if (this.search) {
                    // load flat by search field
                    fetchData = await this.loadSearchFlat();
                } else if (this.group && !this.flags.isOpenedType) {
                    // load Types
                    fetchData = await this.loadTypes();
                } else {
                    // load flats
                    fetchData = await this.loadFlats();
                }

                this.prepareUpsalesInfo(fetchData);

                const objects = [...this.slicedObjects, ...fetchData.results];

                const {
                    initialObjects,
                    mixedObjects,
                    slicedObjects,
                } = this.mixUpsales(objects);

                this.service.offset += this.service.limit;
                this.slicedObjects = slicedObjects;

                if (this.service.count > this.service.offset || this.slicedObjects.length) {
                    this.flags.allowScrollLoad = true;
                }

                stop = Date.now();

                if (stop - start < 400) {
                    setTimeout(() => {
                        this.loadObjectsFromData(mixedObjects, initialObjects);
                    }, 400 - (stop - start));
                } else {
                    this.loadObjectsFromData(mixedObjects, initialObjects);
                }
            },

            loadObjectsFromData(objects, initialObjects, isConcat = false) {
                if (!objects) return;

                if (this.group && !this.flags.isOpenedType) {
                    this.types = isConcat ? this.types.concat(objects) : objects.slice();
                    this.initialTypes = isConcat ? this.initialTypes.concat(initialObjects) : initialObjects.slice();
                } else {
                    this.flats = isConcat ? this.flats.concat(objects) : objects.slice();
                    this.initialFlats = isConcat ? this.initialFlats.concat(initialObjects) : initialObjects.slice();
                }

                setTimeout(() => this.flags.isReloading = false, 400);

            },

            updateObjects: function () {
                this.service.offset = 0;
                this.allowScrollLoad = false;

                if (!this.defaultQueryState) {
                    window.history.replaceState(null, null, this.baseUrlString);
                }

                return this.getObjects();
            },

            loadType: function () {
                let param = this.queryStringify(this.values);
                let url = `${this.routes.typeFilterUrl}${this.controls.type}/?limit=${this.service.limit}&offset=${this.service.offset}&`;

                param = param.split('&').filter(item => item.indexOf('null') === -1).join('&');

                for (let option in this.controls) {
                    if (this.controls[option]) {
                        url += `${option}=${this.controls[option]}&`;
                    }
                }

                if (param) {
                    url += param;
                }

                return fetch(url, {
                    credentials: 'same-origin'
                }).then(response => {
                    return response.json();
                });
            },

            loadSearchFlat: function () {
                let url = `${this.routes.flatFilterUrl}?limit=${this.service.limit}&offset=${this.service.offset}&article=${this.search}`;

                return fetch(url, {
                    credentials: 'same-origin'
                }).then(response => response.json());
            },

            loadTypes: function () {
                let url = `${this.routes.typeFilterUrl}?limit=${this.service.limit - this.slicedObjects.length}&offset=${this.service.offset}&`;
                let param = this.queryStringify(this.values);

                param = param.split('&').filter(item => item.indexOf('null') === -1).join('&');


                for (let option in this.controls) {
                    if (this.controls[option]) {
                        url += `${option}=${this.controls[option]}&`;
                    }
                }

                if (param) {
                    url += param;
                }

                return fetch(url, {
                    credentials: 'same-origin'
                }).then(response => response.json());
            },

            loadFlats: function () {
                let url = `${this.routes.flatFilterUrl}?limit=${this.service.limit - this.slicedObjects.length}&offset=${this.service.offset}&`;
                let param = this.queryStringify(this.values);

                param = param.split('&').filter(item => item.indexOf('null') === -1).join('&');

                for (let option in this.controls) {
                    if (this.controls[option]) {
                        url += `${option}=${this.controls[option]}&`;
                    }
                }
                if (param) {
                    url += param;
                }
                for (let option in this.additionalFilters) {
                    if (this.additionalFilters[option]) {
                        url+=(`&${option}=${this.additionalFilters[option]}`);
                    }
                }
                this.additionalFilters={};
                return fetch(url, {
                    credentials: 'same-origin'
                })
                    .then(response => response.json())
                    .then(data => {

                        if (!this.sentCriteo) {
                            this.sentCriteo = true;
                            const allowedProjectSlugs = ['prichal', 'nagat', 'stresh'];
                            let flatIds = [];
                            let mindboxFlats = data.results.map(el => Object.assign({}, {
                                product: {
                                    ids: {
                                        crmsite: el.crm_site_id,
                                    },
                                },
                                count: '1',
                                pricePerItem: el.price.toString(),
                            }));

                            // первая загрузка страницы, updatedValue.project - массив
                            if (Array.isArray(this.updatedValue.project) && this.updatedValue.project.length) {
                                if (this.updatedValue.project.some(item => allowedProjectSlugs.includes(item))) {
                                    flatIds = data.results.filter(flat => allowedProjectSlugs.includes(flat.project_slug)).map(flat => flat.ref_id);
                                }
                            // если есть изменения в фильтре проекта, updatedValue.project - строка
                            } else if (this.updatedValue.project && allowedProjectSlugs.includes(this.updatedValue.project)) {
                                flatIds = data.results.filter(flat => allowedProjectSlugs.includes(flat.project_slug)).map(flat => flat.ref_id);
                            }
                            if (flatIds.length) {
                                window.criteo_q = window.criteo_q || [];
                                var deviceType = /iPad/.test(navigator.userAgent) ? "t" : /Mobile|iP(hone|od)|Android|BlackBerry|IEMobile|Silk/.test(navigator.userAgent) ? "m" : "d";
                                window.criteo_q.push(
                                    {event: "setAccount", account: 82111}, // Это значение остается неизменным
                                    {event: "setEmail", email: ""},
                                    {event: "setSiteType", type: deviceType},
                                    {event: "viewList", item: flatIds}
                                );
                            }
                            if (mindboxFlats.length) {
                                sendFlats(mindboxFlats);
                            }
                        }

                        return data;
                })
            },

            queryStringify: function (obj) {
                let stringifiedParams = '';

                for (let name in obj) {
                    if (obj[name].length) {
                        if (this.filterTypes[name] === 'range') {
                            if (obj[name][0]) stringifiedParams += `${name}_0=${obj[name][0]}&`;
                            if (obj[name][1]) stringifiedParams += `${name}_1=${obj[name][1]}&`;
                        } else {
                            if (obj[name][0] !== null) {
                                stringifiedParams += `${name}=`;
                                obj[name].forEach(val => {
                                    stringifiedParams += `${val},`
                                });
                                stringifiedParams = stringifiedParams.slice(0, -1) + '&';
                            }
                        }
                    }
                }

                return stringifiedParams.slice(0, -1);
            },

            queryParse: function (query) {
                let parsedParams = {};
                let queryString = query.substring(1);

                if (queryString) {
                    let params = queryString.split('&');

                    params.forEach(param => {
                        let name = param.split('=')[0];
                        let value = param.split('=')[1];

                        if (value) {
                            if (name.match(/(_0|_1)$/)) {
                                name = name.slice(0, -2);
                            }

                            if (!parsedParams.hasOwnProperty(name)) {
                                parsedParams[name] = [];
                            }

                            value.split(',').forEach(val => {
                                parsedParams[name].push(val);
                            });
                        }
                    });
                }

                return parsedParams;
            },

            onSubmitFilter: function () {
                this.$root.$emit('filter-submit');
                this.flags.isOpenedFilter = false;
            },

            onFilterReset(isResetSearch = true) {
                const values = Object.keys(this.values).reduce((acc, key) => {
                    acc[key] = [];
                    return acc;
                }, {});

                if (isResetSearch) {
                    this.search = '';
                }

                this.$root.$emit('update-values', {...values});
                this.$root.$emit('filter-submit');
            },

            mobileScrollTop() {
                if (this.$mq !== 'mobile') return;

                Utils.scrollElement(document.documentElement, 0)
            },

            onLoadType() {
                this.flags.isReloading = true;
                this.flags.allowScrollLoad = false;

                this.loadType()
                    .then(response => {
                        this.type = response;
                        this.flags.isOpenedType = true;
                        Utils.scrollElement(document.documentElement, 0);

                        this.updateObjects()
                            .then(() => {
                                this.flags.isReloading = false;
                                this.flags.allowScrollLoad = true;
                            });
                    })
                    .catch((reason) => {
                        this.type = {};
                        this.flags.isOpenedType = false;
                        throw reason
                    })
                    .finally(() => {
                        // setTimeout(() => {
                        this.flags.isReloading = false;
                        this.flags.allowScrollLoad = true;
                        // }, 400);
                    })
            },

            onUserAction() {
                if (!this.isUserAction) {
                    this.isUserAction = true;
                    window.fbq && window.fbq('track', 'Search');
                }
            },

            prepareUpsalesInfo(data) {
                if (!data) {
                    this.upsalesInfo = {};
                    return;
                }

                const articles = data.articles || [];
                const priceExtend = data.price_extend || {};
                const rareText = data.rare_format_instead_of_price_extend || {};
                const parkingUrl = data.parking_url || null;
                const commercialUrl = data.commercial_url || null;
                const commercialText = data.upsale_text_commercial || null;
                const roomsExtend = data.rooms_extend || null;
                const projectFinished = data.project_finished || null;
                const project_name = data.project_name || null;
                const phaseIsFinished = data.phase_is_finished || {};
                const penthouse = data.penthouse_url || null;
                const nextProject = data.link_to_project_similar_price || null;

                this.upsalesInfo = {
                    articles,
                    priceExtend,
                    parkingUrl,
                    commercialUrl,
                    commercialText,
                    roomsExtend,
                    projectFinished,
                    project_name,
                    rareText,
                    phaseIsFinished,
                    penthouse,
                    nextProject,
                };
            },

            mixUpsales(objectsList) {
                let objects = [];
                let slicedObjects = [];
                let initialSlicedObjects = [];
                let upsales = [];

                if (objectsList && objectsList.length) {

                    let possibleUpsalesCount;

                    objects = [...objectsList];

                    if (this.service.offset === 0) {
                        possibleUpsalesCount = 1;
                        // if (objectsList.length > this.service.limit) {
                        //     // possibleUpsalesCount = Math.floor(objectsList.length / this.service.limit * 2) - 1;
                        // } else {
                        //     // possibleUpsalesCount = 1
                        // }
                    } else {
                        // possibleUpsalesCount = 2;
                        possibleUpsalesCount = Math.floor((objectsList.length + 1) / this.service.limit * 2);
                        // possibleUpsalesCount = Math.floor(objectsList.length / this.service.limit * 2);
                    }

                    if (possibleUpsalesCount) {
                        new Array(possibleUpsalesCount)
                            .fill('')
                            .forEach(() => {
                                let upsale = null;

                                if (this.service.lastStaticUpsaleIndex < upsaleCardTypes.length) {
                                    upsale = this.getStaticUpsale(upsaleCardTypes);
                                }

                                if (!upsale) {
                                    upsale = this.getCustomerUpsale();
                                }

                                if (!upsale) return;

                                upsales.push(upsale);
                            });
                    }

                    if (upsales.length && upsales.length + objects.length > this.service.limit) {
                        initialSlicedObjects = objects.slice(0, upsales.length * -1);
                        slicedObjects = objects.splice(upsales.length * -1);
                    } else {
                        initialSlicedObjects = [...objectsList];
                    }

                    upsales.forEach((upsale) => {
                        const index = this.getUpsaleIndex(this.service.upsalesCount, this.service.limit);

                        objects.splice(index, 0, upsale);
                        this.service.upsalesCount += 1;
                    });
                }

                return {
                    initialObjects: initialSlicedObjects,
                    mixedObjects: objects,
                    slicedObjects,
                }
            },

            getStaticUpsale(upsaleTypes) {
                // upsaleTypes: ['priceIncrease', 'parking', 'roomIncrease', 'callback', 'commercial'];
                const upsalesLeft = upsaleTypes.slice(this.service.lastStaticUpsaleIndex);

                for (let i = 0; i < upsalesLeft.length; i++) {
                    const upsaleType = upsalesLeft[i];

                    this.service.lastStaticUpsaleIndex += 1;

                    if (upsaleType === 'priceIncrease') {
                        if (!this.upsalesInfo.priceExtend || !this.upsalesInfo.priceExtend.count || this.upsalesInfo.priceExtend.count <= 5) continue;
                        return this.getIncreaseUpsale();
                    }

                    if (upsaleType === 'parking') {
                        if (!this.upsalesInfo.parkingUrl) continue;
                        return this.getParkingUpsale();
                    }

                    if (upsaleType === 'roomIncrease' && (!this.upsalesInfo.rareText || !this.upsalesInfo.rareText[0])) {
                        if (!this.upsalesInfo.roomsExtend || !this.upsalesInfo.roomsExtend.url || !this.upsalesInfo.roomsExtend.available) continue;
                        return this.getRoomIncreaseUpsale();
                    }

                    if (upsaleType === 'rareText') {
                        if (this.upsalesInfo.rareText && this.upsalesInfo.rareText[0])
                            return this.getRareText(this.upsalesInfo.rareText[0]);
                    }

                    if (upsaleType === 'callback') {
                        return {
                            key: 'callback',
                            type: 'callback'
                        };
                    }

                    if (upsaleType === 'commercial') {
                        if (!this.upsalesInfo.commercialUrl) continue;

                        return this.getCommercialUpsale();
                    }

                    if (upsaleType === 'projectFinish') {
                        if (!this.upsalesInfo.projectFinished) continue;

                        return this.getProjectFinishUpsale();
                    }

                    if (upsaleType === 'phaseIsFinished') {
                        if (!this.upsalesInfo || !this.upsalesInfo.phaseIsFinished || !Object.keys(this.upsalesInfo.phaseIsFinished).length) continue;

                        return this.getPhaseIsFinishedUpsale();
                    }
                    
                    if (upsaleType === 'penthouse') {
                        if (!this.upsalesInfo.penthouse ) continue;
                        
                        return this.getPenthouseUpsale();
                    }

                    if (upsaleType === 'nextProject') {
                      if (!this.upsalesInfo.nextProject) continue;

                      return this.getNextProjectUpsale();
                    }
                }

                return null;
            },

            getRareText(upsale) {
                const proj = this.upsalesInfo.project_name;

                return this.generateUpsaleObject({
                    key: `customer-upsale-rare-text`,
                    isContent: true,
                    isCustomer: true,
                    isBigImage: upsale.has_big_image,
                    title: upsale.title,
                    image: upsale.image,
                    description: upsale.description,
                    buttonText: upsale.button_text,
                    typeName: proj,
                    dataAttributes: [{name: 'data-url', value: `/api/material/${upsale.material}/`}],
                });
            },

            getCustomerUpsale() {
                const index = this.service.lastCustomerUpsaleIndex;
                const upsale = this.upsalesInfo.articles[index];

                if (!upsale) return null;

                this.service.lastCustomerUpsaleIndex += 1;

                return this.generateUpsaleObject({
                    key: `customer-upsale-${upsale.id}`,
                    isContent: true,
                    isCustomer: true,
                    isBigImage: upsale.has_big_image,
                    title: upsale.title,
                    image: upsale.image,
                    description: upsale.description,
                    buttonText: upsale.button_text,
                    typeName: 'Статья',
                    dataAttributes: [{name: 'data-url', value: `/api/material/${upsale.material}/`}],
                });
            },

            getIncreaseUpsale() {
                const count = this.upsalesInfo.priceExtend.count;
                const url = this.upsalesInfo.priceExtend.url;

                return this.generateUpsaleObject({
                    key: 'upsalePriceIncrease',
                    title: `${count}`,
                    image: '/static/realty/upsale/upsalePriceIncrease.jpg',
                    isBigImage: true,
                    description: 'Вы можете увидеть больше вариантов, если увеличите свой бюджет на 300 000 рублей',
                    buttonText: 'Посмотреть',
                    typeName: 'Позвольте себе больше',
                    url
                })
            },

            getParkingUpsale() {
                const url = this.upsalesInfo.parkingUrl;
                const proj = this.upsalesInfo.project_name;

                return this.generateUpsaleObject({
                    key: 'upsaleParking',
                    title: 'Паркинг',
                    image: '/static/realty/upsale/parking-n.png',
                    description: 'Во дворах комплексов Level&nbsp;Group нет машин. Выберите место для вашего автомобиля в подземном паркинге.',
                    buttonText: 'Выбрать паркинг',
                    typeName: proj,
                    url
                });
            },

            getRoomIncreaseUpsale() {
                const url = this.upsalesInfo.roomsExtend.url;
                const roomValue = this.upsalesInfo.roomsExtend.value;

                return this.generateUpsaleObject({
                    key: 'upsaleRoomIncrease',
                    title: 'Расширьтесь',
                    image: '/static/realty/upsale/expand.png',
                    description: 'За вашу стоимость можно посмотреть выгодные предложения большей комнатности.',
                    buttonText: `${roomValue}-комнатные варианты`,
                    typeName: 'Выгодное предложение',
                    url,
                });
            },

            getCommercialUpsale() {
                const url = this.upsalesInfo.commercialUrl;
                const proj = this.upsalesInfo.project_name;
                const text = this.upsalesInfo.commercialText;

                return this.generateUpsaleObject({
                    key: 'upsaleCommercial',
                    title: 'Коммерческие помещения',
                    image: '/static/realty/upsale/commercial.png',
                    description: text ? text : 'Коммерческие помещения на&nbsp;этапе строительства по&nbsp;инвестиционно привлекательной цене.',
                    buttonText: `Выбрать помещение`,
                    typeName: proj,
                    url,
                });
            },

            getProjectFinishUpsale() {
                const background = this.upsalesInfo.projectFinished.upsale_img_parking;

                return this.generateUpsaleObject({
                    key: 'upsaleProjectFinish',
                    backgroundImage: background,
                    centralTitle: 'Дом сдан!',
                    centralSubtitle: 'Ключи в момент оплаты!',
                    isBigImage: true,
                });
            },

            getPhaseIsFinishedUpsale() {
                const background = this.upsalesInfo.phaseIsFinished.background_upsale_display;
                const title = this.upsalesInfo.phaseIsFinished.title ;
                const subtitle = this.upsalesInfo.phaseIsFinished.subtitle;

                return this.generateUpsaleObject({
                    key: 'upsalePhaseIsFinished',
                    backgroundImage: background,
                    isBigImage: true,
                    finishedTitle: title,
                    finishedSubtitle: subtitle,
                })

            },
            
            getPenthouseUpsale() {
                const upsale = {
                    button_text: 'Читать',
                    description: 'Эксклюзивные пентхаусы на&nbsp;25-м этаже с&nbsp;террасами',
                    has_big_image: true,
                    id: 100,
                    image: '/static/realty/upsale/stresh-penthouse.jpg',
                    material: 21,
                    title: 'Cобственный пентхаус',
                }

                if (!upsale) return null;

                return this.generateUpsaleObject({
                    key: `customer-upsale-${upsale.id}`,
                    isContent: true,
                    isCustomer: true,
                    isBigImage: upsale.has_big_image,
                    title: upsale.title,
                    image: upsale.image,
                    description: upsale.description,
                    buttonText: upsale.button_text,
                    typeName: 'Статья',
                    dataAttributes: [{name: 'data-url', value: `/api/material/${upsale.material}/`}],
                });
            },

            getNextProjectUpsale() {
                const projectName = this.upsalesInfo.nextProject.name;
                const backgroundImage = this.upsalesInfo.nextProject.upsale_img_to_project;
                const text = this.upsalesInfo.nextProject.card_text;
                const minPrice = this.upsalesInfo.nextProject.min_price;
                const url = this.upsalesInfo.nextProject.url;

                const upsale = {
                    projectName,
                    backgroundImage,
                    text,
                    minPrice,
                    url,
                }

                if (!upsale) return null;

                return this.generateUpsaleObject({
                    key: 'upsaleNextProject',
                    typeName: 'Проекты',
                    title: projectName,
                    backgroundImage,
                    isBigImage: true,
                    description: text,
                    price: minPrice.toFixed(2),
                    buttonText: 'Перейти',
                    url,
                    urlTarget: false,
                })
            },

            generateUpsaleObject(options = {}) {
                const isBigImage = options.isBigImage || false;
                const isCustomer = options.isCustomer || false;
                const isContent = options.isContent || false;
                const imageUrl = options.image || '';
                const title = options.title || '';
                const description = options.description || '';
                const buttonText = options.buttonText || '';
                const url = options.url || '';
                const actionClass = options.actionClass || '';
                const dataAttributes = options.dataAttributes || [];
                const backgroundImage = options.backgroundImage || '';
                const centralTitle = options.centralTitle || '';
                const centralSubtitle = options.centralSubtitle || '';
                const typeName = options.typeName || false;
                const finishedTitle = options.finishedTitle || '';
                const finishedSubtitle = options.finishedSubtitle || '';
                const price = options.price || '';
                const urlTarget = options.urlTarget || '';

                return {
                    type: 'upsale',
                    image: imageUrl,
                    key: options.key,
                    isBigImage,
                    isContent,
                    isCustomer,
                    title,
                    description,
                    buttonText,
                    url,
                    actionClass,
                    dataAttributes,
                    backgroundImage,
                    centralTitle,
                    centralSubtitle,
                    typeName,
                    finishedTitle,
                    finishedSubtitle,
                    price,
                    urlTarget,
                }
            },

            getUpsaleIndex(upsalesCount, limit) {
                if (upsalesCount === 0) return limit;

                const isEven = upsalesCount % 2 === 0;

                return isEven ? limit - 1 : 1;
                // return isEven ? upsalesCount * limit + 2 : (upsalesCount + 1) * limit
            },

            resetAlterRoom() {
                if (this.values.alter_room) {
                    this.values.alter_room = [];
                }
            },

            resetUpsales() {
                this.upsalesInfo = {};
                this.service.upsalesCount = 0;
                this.service.lastStaticUpsaleIndex = 0;
                this.service.lastCustomerUpsaleIndex = 0;
                this.slicedObjects = [];
            },

            pushMindboxEvent(operationName = '') {
                if (!operationName) return;

                eventPush(operationName);
            },

            fixTitle() {
                if (this.values.project.includes('amur') || this.values.project.includes(null)) {
                    title.textContent = 'Квартиры и апартаменты';
                } else if (this.values.project.includes('stresh') || this.values.project.includes('don')) {
                    title.textContent = 'Апартаменты';
                } else {
                    title.textContent = 'Квартиры';
                }
            }
        }
    }
</script>
