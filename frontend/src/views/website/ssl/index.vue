<template>
    <div>
        <RouterButton :buttons="routerButton" />
        <LayoutContent :title="$t('website.ssl')">
            <template #prompt>
                <el-alert type="info" :closable="false">
                    <template #default>
                        <span><span v-html="$t('website.encryptHelper')"></span></span>
                    </template>
                </el-alert>
            </template>
            <template #toolbar>
                <el-button type="primary" @click="openSSL()">
                    {{ $t('ssl.create') }}
                </el-button>
                <el-button type="primary" @click="openUpload()">
                    {{ $t('ssl.upload') }}
                </el-button>
                <el-button type="primary" plain @click="openAcmeAccount()">
                    {{ $t('website.acmeAccountManage') }}
                </el-button>
                <el-button type="primary" plain @click="openDnsAccount()">
                    {{ $t('website.dnsAccountManage') }}
                </el-button>
            </template>
            <template #main>
                <br />
                <ComplexTable :data="data" :pagination-config="paginationConfig" @search="search()" v-loading="loading">
                    <el-table-column
                        :label="$t('website.domain')"
                        fix
                        show-overflow-tooltip
                        prop="primaryDomain"
                    ></el-table-column>
                    <el-table-column
                        :label="$t('website.otherDomains')"
                        fix
                        show-overflow-tooltip
                        prop="domains"
                    ></el-table-column>
                    <el-table-column
                        :label="$t('ssl.applyType')"
                        fix
                        show-overflow-tooltip
                        prop="provider"
                        width="100px"
                    >
                        <template #default="{ row }">{{ getProvider(row.provider) }}</template>
                    </el-table-column>
                    <el-table-column
                        :label="$t('ssl.acmeAccount')"
                        fix
                        show-overflow-tooltip
                        prop="acmeAccount.email"
                    ></el-table-column>
                    <el-table-column
                        :label="$t('commons.table.status')"
                        fix
                        show-overflow-tooltip
                        prop="status"
                        width="100px"
                    >
                        <template #default="{ row }">
                            <el-popover
                                v-if="
                                    row.status === 'error' ||
                                    row.status === 'applyError' ||
                                    row.status === 'systemRestart'
                                "
                                placement="bottom"
                                :width="400"
                                trigger="hover"
                                :content="row.message"
                            >
                                <template #reference>
                                    <Status :key="row.status" :status="row.status"></Status>
                                </template>
                            </el-popover>
                            <div v-else>
                                <Status :key="row.status" :status="row.status"></Status>
                            </div>
                        </template>
                    </el-table-column>
                    <el-table-column :label="$t('website.log')" width="100px">
                        <template #default="{ row }">
                            <el-button @click="openLog(row)" link type="primary">{{ $t('website.check') }}</el-button>
                        </template>
                    </el-table-column>
                    <el-table-column
                        :label="$t('website.brand')"
                        fix
                        show-overflow-tooltip
                        prop="organization"
                    ></el-table-column>
                    <el-table-column :label="$t('ssl.autoRenew')" fix width="100px">
                        <template #default="{ row }">
                            <el-switch
                                :disabled="row.provider === 'dnsManual' || row.provider === 'manual'"
                                v-model="row.autoRenew"
                                @change="updateConfig(row)"
                            />
                        </template>
                    </el-table-column>
                    <el-table-column
                        prop="expireDate"
                        :label="$t('website.expireDate')"
                        :formatter="dateFormat"
                        show-overflow-tooltip
                    />
                    <fu-table-operations
                        :ellipsis="3"
                        :buttons="buttons"
                        :label="$t('commons.table.operate')"
                        :fixed="mobile ? false : 'right'"
                        fix
                    />
                </ComplexTable>
            </template>
            <DnsAccount ref="dnsAccountRef"></DnsAccount>
            <AcmeAccount ref="acmeAccountRef"></AcmeAccount>
            <Create ref="sslCreateRef" @close="search()"></Create>
            <Detail ref="detailRef"></Detail>
            <SSLUpload ref="sslUploadRef" @close="search()"></SSLUpload>
            <Apply ref="applyRef" @search="search" />
            <OpDialog ref="opRef" @search="search" />
            <Log ref="logRef" @close="search()" />
        </LayoutContent>
    </div>
</template>

<script lang="ts" setup>
import { onMounted, reactive, ref, computed } from 'vue';
import OpDialog from '@/components/del-dialog/index.vue';
import { DeleteSSL, ObtainSSL, SearchSSL, UpdateSSL } from '@/api/modules/website';
import DnsAccount from './dns-account/index.vue';
import AcmeAccount from './acme-account/index.vue';
import Create from './create/index.vue';
import Detail from './detail/index.vue';
import { dateFormat, getProvider } from '@/utils/util';
import i18n from '@/lang';
import { Website } from '@/api/interface/website';
import { MsgSuccess } from '@/utils/message';
import { GlobalStore } from '@/store';
import SSLUpload from './upload/index.vue';
import Apply from './apply/index.vue';
import Log from '@/components/log-dialog/index.vue';

const globalStore = GlobalStore();
const paginationConfig = reactive({
    cacheSizeKey: 'ssl-page-size',
    currentPage: 1,
    pageSize: 10,
    total: 0,
});
const acmeAccountRef = ref();
const dnsAccountRef = ref();
const sslCreateRef = ref();
const detailRef = ref();
const data = ref();
const loading = ref(false);
const opRef = ref();
const sslUploadRef = ref();
const applyRef = ref();
const logRef = ref();

const routerButton = [
    {
        label: i18n.global.t('website.ssl'),
        path: '/websites/ssl',
    },
];

const buttons = [
    {
        label: i18n.global.t('ssl.detail'),
        disabled: function (row: Website.SSLDTO) {
            return row.status === 'init' || row.status === 'error';
        },
        click: function (row: Website.SSLDTO) {
            openDetail(row.id);
        },
    },
    {
        label: i18n.global.t('ssl.apply'),
        disabled: function (row: Website.SSLDTO) {
            return row.status === 'applying';
        },
        click: function (row: Website.SSLDTO) {
            if (row.provider === 'dnsManual') {
                applyRef.value.acceptParams({ ssl: row });
            } else {
                applySSL(row);
            }
        },
    },
    {
        label: i18n.global.t('commons.button.delete'),
        click: function (row: Website.SSLDTO) {
            deleteSSL(row);
        },
    },
];

const mobile = computed(() => {
    return globalStore.isMobile();
});

const search = () => {
    const req = {
        page: paginationConfig.currentPage,
        pageSize: paginationConfig.pageSize,
    };
    loading.value = true;
    SearchSSL(req)
        .then((res) => {
            data.value = res.data.items || [];
            paginationConfig.total = res.data.total;
        })
        .finally(() => {
            loading.value = false;
        });
};

const updateConfig = (row: Website.SSLDTO) => {
    loading.value = true;
    UpdateSSL({ id: row.id, autoRenew: row.autoRenew })
        .then(() => {
            MsgSuccess(i18n.global.t('commons.msg.updateSuccess'));
        })
        .finally(() => {
            loading.value = false;
        });
};

const openAcmeAccount = () => {
    acmeAccountRef.value.acceptParams();
};
const openDnsAccount = () => {
    dnsAccountRef.value.acceptParams();
};
const openSSL = () => {
    sslCreateRef.value.acceptParams();
};
const openUpload = () => {
    sslUploadRef.value.acceptParams();
};
const openDetail = (id: number) => {
    detailRef.value.acceptParams(id);
};
const openLog = (row: Website.SSLDTO) => {
    logRef.value.acceptParams({ id: row.id, type: 'ssl' });
};

const applySSL = (row: Website.SSLDTO) => {
    loading.value = true;
    ObtainSSL({ ID: row.id })
        .then(() => {
            MsgSuccess(i18n.global.t('ssl.applyStart'));
            search();
        })
        .finally(() => {
            loading.value = false;
        });
};

const deleteSSL = async (row: any) => {
    opRef.value.acceptParams({
        title: i18n.global.t('commons.button.delete'),
        names: [row.primaryDomain],
        msg: i18n.global.t('commons.msg.operatorHelper', [
            i18n.global.t('website.ssl'),
            i18n.global.t('commons.button.delete'),
        ]),
        api: DeleteSSL,
        params: { id: row.id },
    });
};

onMounted(() => {
    search();
});
</script>
