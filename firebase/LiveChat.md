This snippet codes below are written in Kotlin ( use Kotlin, okay! :') )

## 1. Import the FirebaseFirestore library
Add the library into your application level build.gradle file and then Sync the Project.
```
implementation 'com.google.firebase:firebase-firestore:21.5.0'
```

## 2. Initialize FirebaseFirestore
We have to initialize the FirebaseFirestore either in Activity or Fragment.
```kotlin
class LiveFragment : BaseFragment() {
    private lateinit var db: FirebaseFirestore

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        rootView = inflater.inflate(R.layout.your_layout, container, false)
        db = FirebaseFirestore.getInstance()
        return rootView
    }
}
```

## 3. Get Live Chat Status First
Live Chat can be either active or not active, so we should check it and listen if there is any changes of the status.
```kotlin
class LiveFragment : BaseFragment() {
    ...
    private var statusChatListener: ListenerRegistration? = null

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val channelId = "1"
        initStatusChatListener(statusChatName = channelId)
    }
    
    private fun initStatusChatListener(statusChatName: String) {
        statusChatListener?.remove()

        val request = db.collection("statusChat").document(statusChatName)
        statusChatListener = request.addSnapshotListener(MetadataChanges.INCLUDE) { snapshot, e ->
            if (e != null) {
                Log.w(TAG, "Listen failed.", e)
                return@addSnapshotListener
            }
            if (snapshot != null && snapshot.metadata.hasPendingWrites()) {    // LOCAL
                /* Do nothing on local snapshot */
                Log.d(TAG, "Local snapshot for status chat. Do nothing.")
            } else {    // SERVER
                val documentData = snapshot?.data
                if (documentData == null) {
                    showOrHideLiveChat(isShow = false)
                    return@addSnapshotListener
                }
                val isActive = documentData["isActive"].toString().toBoolean()
                if (isActive) {
                    showOrHideLiveChat(isShow = true)
                } else {
                    showOrHideLiveChat(isShow = false)
                }
            }
        }
    }
}
```

## 4. Get 10 Newest Chats First
We want to get 10 newest on first load, just load once and don't using SnapshotListener to avoid Firestore quota wasted. We limit the document with only 10 files, with `db.collection().limit(10)`.
```kotlin
class LiveFragment : BaseFragment() {
    ...
    private var chatListener: ListenerRegistration? = null
    private var liveChatModelList: MutableList<LiveChatModel>? = null
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val chatChannelId = "chat1"
        initChatListener(chatName = chatChannelId)
    }
    
    private fun initChatListener(chatName: String) {
        chatListener?.remove()

        db.collection(chatName)
                .orderBy("ts", Query.Direction.DESCENDING)
                .limit(10)
                .get()
                .addOnSuccessListener { result ->
                    when {
                        result == null || result.isEmpty -> {
                            Log.d(TAG, "Empty document from collection $chatName")
                        }
                        else -> {
                            if (liveChatModelList == null) {
                                liveChatModelList = ArrayList()
                            }
                            for (document in result) {
                                val userName = document["u"].toString()
                                val timeMillis = document["ts"].toString()
                                val profilePicture = document["i"].toString()
                                val message = document["m"].toString()
                                
                                /* Insert into your ArrayList chat model. */
                                liveChatModelList!!.add(0, LiveChatModel(userName, timeMillis, profilePicture, message))
                            }
                            liveChatAdapter.setData(liveChatModelList!!)
                        }
                    }
                    initRealtimeChatListener(chatName)
                }
                .addOnFailureListener { exception ->
                    Log.w(TAG, "Error getting documents", exception)
                    when {
                        chatName != getChatRoomName(channelId) -> {
                            Log.d(TAG, "chatName has been changed! Previous was $chatName and now is ${getChatRoomName(channelId)}")
                        }
                        else -> initRealtimeChatListener(chatName)
                    }
                }
    }
}
```
## 5. Listen to New Chats
After we load 10 newest chats, then we want to listen if there is a new chat inserted, so we limit with only 1 document. When you start to listen, Firestore will return the first data with the the newest chat from 10 chats before, so we want to skip that chat. I use Boolean variable of `isFirstRequestSnapshot` to check that first document.
```kotlin
class LiveFragment : BaseFragment() {
    ...
    private var isFirstRequestSnapshot = true
    private var chatListener: ListenerRegistration? = null
    private var liveChatModelList: MutableList<LiveChatModel>? = null
    
    private fun initChatListener(chatName: String) {
        ...
        initRealtimeChatListener(chatName)
    }
    
    private fun initRealtimeChatListener(chatName: String) {
        chatListener?.remove()
        isFirstRequestSnapshot = true

        val realtimeRequest = db.collection(chatName)
                .orderBy(ChatColumn.TIME_MILLIS.columnName, Query.Direction.DESCENDING)
                .limit(1)

        chatListener = realtimeRequest.addSnapshotListener { value, e ->
            if (e != null) {
                Log.w(TAG, "Listen failed.", e)
                return@addSnapshotListener
            }
            when {
                value == null || value.isEmpty -> {
                    Log.d(TAG, "Empty document from collection $chatName")
                }
                else -> {
                    if (liveChatModelList == null) {
                        liveChatModelList = ArrayList()
                    }
                    for (document in value) {
                        val userName = document["u"].toString()
                        val timeMillis = document["ts"].toString()
                        val profilePicture = document["i"].toString()
                        val message = document["m"].toString()

                        if (isFirstRequestSnapshot) {
                            isFirstRequestSnapshot = false
                            if (Util.isNotNull(liveChatModelList)) {
                                val lastChatModel = liveChatModelList!![liveChatModelList!!.size - 1]
                                if (lastChatModel.time == timeMillis) {
                                    continue
                                }
                            }
                        }
                        /* Insert into your ArrayList chat model. */
                        liveChatModelList!!.add(LiveChatModel(userName, timeMillis, profilePicture, message))
                    }
                    liveChatAdapter.setData(liveChatModelList!!)
                }
            }
        }
    }
}
```
