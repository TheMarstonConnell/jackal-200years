<script setup lang="ts">
import { ref, type Ref } from 'vue'
import 'bytemd/dist/index.css'
import { type IWalletHandler, WalletHandler, type IFileIo, FileIo, FileUploadHandler, getFileTreeData } from '@jackallabs/jackal.js'
import type { IUploadList } from '@jackallabs/jackal.js'

type FileData = {
  name: string;
  fid: string;
};

const wallet: Ref<IWalletHandler | null> = ref(null)
const walletActive: Ref<boolean> = ref(false)
const fileIo: Ref<IFileIo | null> = ref(null)
const data: Ref<FileData[]> = ref([])
const singleFile: Ref<HTMLElement | null> = ref(null)

const path = "200years"

const mSignerChain = 'jackal-1'
const mainnet = {
  signerChain: mSignerChain,
  enabledChains: [mSignerChain],
  queryAddr: 'https://grpc.jackalprotocol.com',
  txAddr: 'https://rpc.jackalprotocol.com',
  chainConfig: { // mainnet chain config
    chainId: mSignerChain,
    chainName: 'Jackal',
    rpc: 'https://rpc.jackalprotocol.com',
    rest: 'https://api.jackalprotocol.com',
    bip44: {
      coinType: 118
    },
    coinType: 118,
    stakeCurrency: {
      coinDenom: 'JKL',
      coinMinimalDenom: 'ujkl',
      coinDecimals: 6
    },
    bech32Config: {
      bech32PrefixAccAddr: 'jkl',
      bech32PrefixAccPub: 'jklpub',
      bech32PrefixValAddr: 'jklvaloper',
      bech32PrefixValPub: 'jklvaloperpub',
      bech32PrefixConsAddr: 'jklvalcons',
      bech32PrefixConsPub: 'jklvalconspub'
    },
    currencies: [
      {
        coinDenom: 'JKL',
        coinMinimalDenom: 'ujkl',
        coinDecimals: 6
      }
    ],
    feeCurrencies: [
      {
        coinDenom: 'JKL',
        coinMinimalDenom: 'ujkl',
        coinDecimals: 6,
        gasPriceStep: {
          low: 0.002,
          average: 0.002,
          high: 0.02
        }
      }
    ],
    features: []
  }
}

async function connectWallet() {

  const walletConfig = {
    selectedWallet: 'keplr',
    signerChain: mainnet.signerChain,
    enabledChains: mainnet.enabledChains,
    queryAddr: mainnet.queryAddr,
    txAddr: mainnet.txAddr,
    chainConfig: mainnet.chainConfig
  }

  // Hooking up the wallet to your app
  wallet.value = await WalletHandler.trackWallet(walletConfig)

  fileIo.value = await FileIo.trackIo(wallet.value, '1.1.2')

  const listOfFolders = [path]
  // you can create as many folders as you would like this way

  // after the first time, this code can be used instead. this will only create new folders if they don't already exist
  await fileIo.value.verifyFoldersExist(listOfFolders)
  walletActive.value = true
  updateFileList()
}

const complete = function () {
  loading.value = false
}

const uploadFile = function (file: File) {
  loading.value = true

  const fileName = file.name
  if (fileName.length == 0) {
    alert("file needs name")
    complete()
    return
  }

  const parentFolderPath = "s/" + path // replace this with your own path

  FileUploadHandler.trackFile(file, parentFolderPath).then((handler) => {
    fileIo.value?.downloadFolder(parentFolderPath).then((parent) => {
      const uploadList: IUploadList = {}
      uploadList[fileName] = {
        data: null,
        exists: false,
        handler: handler,
        key: fileName,
        uploadable: handler.getForPublicUpload()
      }

      fileIo.value?.staggeredUploadFiles(uploadList, parent, {
        complete: 0,
        timer: 0
      }).then(() => {
        if (wallet.value == null) {
          complete()
          return
        }

        getFileTreeData("s/" + path + "/" + fileName, wallet.value.getJackalAddress(), wallet.value.getQueryHandler()).then(f => {
          const fFiles = f.value.files
          if (fFiles == null) {
            complete()
            return
          }
          const fidList = JSON.parse(fFiles.contents)
          const newFid = fidList.fids[0]
          console.log(newFid)

          updateFileList()
          complete()
        }).catch(complete)
      }).catch(complete)
    }).catch(complete)
  }).catch(complete)


}

const openFile = async function (fileName: string) {
  if (wallet.value == null) {
    return
  }
  const link = "https://jackal.link/p/" + wallet.value.getJackalAddress() + "/" + path + "/" + fileName
  const w = window.open(link, '_blank')
  if (w == null) {
    return
  }
  w.focus();
}


const loading = ref(false)

const isDisabled = function (): boolean {
  return !walletActive.value || loading.value
}

async function updateFileList() {
  if (fileIo.value == null) {
    return
  }
  if (wallet.value == null) {
    return
  }
  const listFiles = await fileIo.value.downloadFolder("s/" + path)
  const files = listFiles.getFolderDetails().fileChildren

  let d = []

  let x = 0
  for (const key of Object.keys(files)) {
    const f = files[key]

    const fDetails = await getFileTreeData("s/" + path + "/" + f.name, wallet.value.getJackalAddress(), wallet.value.getQueryHandler())
    console.log(fDetails)
    const dFiles = fDetails.value.files
    if (dFiles == null) {
      continue
    }
    const fidList = JSON.parse(dFiles.contents)
    const newFid = fidList.fids[0]

    d[x] = { name: key, fid: newFid }
    x++
  }
  console.log(d)
  data.value = d
}

async function addSingles(e: any) {
  const { files } = e.target
  const file = files[0]
  uploadFile(file)
}

</script>

<template >
  <main>
    <h1>Jackal 200 Year Uploader</h1>
    <button type="button" @click="connectWallet">{{ (walletActive == true) ? "Connected" : "Connect Wallet" }}</button>
    <input id="fileUpload" ref="singleFile" type="file" :disabled="isDisabled()" @change="addSingles" />
    <div class="files">
      <div class="card" v-for="item in data" :key="item.name" @click="function () { openFile(item.name) }">
        <h1 class="file-title">{{ item.name }}</h1>
      </div>
    </div>
  </main>
</template>

<style>
#app {
  max-width: 1280px;
  margin: 0 auto;
  font-weight: normal;
  /* padding-top: 10px; */
}

h1 {
  margin-top: 0px;
}

.bytemd {
  width: 60vw;
  height: calc(100vh - 200px);
  margin-left: auto;
  margin-right: auto;
  margin-bottom: 30px;
  margin-top: 30px;
}



.file-title:hover {
  text-decoration: underline;
}

button {
  margin-left: 20px;
  margin-right: 20px;
  border-width: 1px;
  border-style: solid;
  border-color: black;
}

button:disabled {
  cursor: auto;
  border-color: black;
}

.filename {
  box-sizing: border-box;
  padding: 5px;
  font-size: 16px;
  border-width: 1px;
  border-radius: 8px;
  height: 40px;
  padding-left: 10px;
}

.bytemd-body {
  text-align: left;
}

.files {
  display: grid;
  grid-auto-flow: row;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 10px 10px;
  margin-top: 30px;
}

.card>h1 {
  font-size: 1.6em;
}

.card:hover>.file-title {
  text-decoration: underline;
}

.card {
  border-radius: 5px;
  border-style: solid;
  padding: 10px;
  overflow: hidden;
  cursor: pointer;
}
</style>
