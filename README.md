### OpenWorldGame

#### 포톤 기반 오픈월드 게임 


#### PlayerController.cs

```C#
using System;
using System.Collections;
using System.Collections.Generic;
using Photon.Pun;
using Unity.VisualScripting;
using UnityEngine;
using UnityEngine.InputSystem;
using Random = UnityEngine.Random;

[RequireComponent(typeof(Rigidbody))]
public class PlayerController : MonoBehaviourPunCallbacks
{
    [SerializeField] float _speed = 100f;
    [SerializeField] float _radius = 1f;
    [SerializeField] float _jumpPower = 0.4f;
    [SerializeField] float _sprintSpeed = 200f;
    // TODO : 추후 SerializeField 제거
    [SerializeField] bool _isGrounded;
    [SerializeField] bool _isSprint;
    [SerializeField] Transform _camTrans;
    [SerializeField] int _damage=10;
    [SerializeField] float _moveAnimationMod = 200f;
    [SerializeField] float _jumpTime = 1f;
    [SerializeField] float _jumpTimer = 0f;
    int _animXSpeed;
    int _animYSpeed;
    int _animJump;
    int _animAttack;
    float _offsetSpeed;
    bool _canJump = true;
    Vector3 _direction;
    Rigidbody _rigidbody;
    Vector3 _moveDir;
    Animator _animator;
    AnimationEventHandler _animationEventHandler;
    Transform _playerBody;
    PhotonView _pv;

    public bool IsEquip;

    void Start()
    {
        Debug.Log("플레이어 생성 !");
        Init();
        AnimInit();
    }
    
    //최초 할당 
    void Init()
    {
        _pv = GetComponent<PhotonView>();
        _animator = GetComponentInChildren<Animator>();
        _rigidbody = GetComponent<Rigidbody>();
        _offsetSpeed = _speed;
        _speed = 0;
        _playerBody = _animator.transform;
        _animationEventHandler = GetComponentInChildren<AnimationEventHandler>();
        _animationEventHandler.AttackEvent += AttackEvent;
    }
    //최초 애니메이션 할당 
    void AnimInit()
    {
        _animXSpeed = Animator.StringToHash("xSpeed");
        _animYSpeed = Animator.StringToHash("ySpeed");
        _animAttack = Animator.StringToHash("randomAttack");
        _animJump = Animator.StringToHash("isJump");
    }

    //서버 접속간 본인 캐릭터만 컨트롤가능하도록 주기적 체크수행
    void Update()
    {
        if (!_pv.IsMine)
            return;
        JumpDelayTimer();
    }


    //서버 접속간 본인 캐릭터만 컨트롤가능하도록 주기적 체크수행 (물리)
    void FixedUpdate()
    {
        if (!_pv.IsMine)
            return;
        Move();
        CheckGround();
    }

    void OnMove(InputValue value)
    {
        Vector2 inputMove = value.Get<Vector2>();
        _direction.x = inputMove.x;
        _direction.z = inputMove.y;
        _speed = Mathf.Approximately(_direction.magnitude, 0f) ? 0f : _offsetSpeed;
    }

    //바닥체크 . . 
    void OnJump()
    {
        if (!_isGrounded || !_canJump)
            return;
        Vector3 forceVel = _rigidbody.velocity;
        forceVel.y = -_jumpPower * Physics.gravity.y;
        _rigidbody.velocity = forceVel;
        _animator.SetBool(_animJump, true);
        _jumpTimer = _jumpTime;
    }

    void OnSprint(InputValue value)
    {
        _isSprint = value.isPressed;
    }

    // TODO : Attack 관련 -> 장착상태일시 스크립터블 참조하여 별도 애니메이션 수행
    void OnFire()
    {
        if (!_pv.IsMine)
            return;
        int randomAttack = Random.Range(1, 3);
        if (!_animator.GetCurrentAnimatorStateInfo(1).IsName("RandomAttack"))
        {
            _animator.SetFloat(_animAttack, randomAttack);
        }
    }

    // 공격 Raycast 기즈모로 확인가능 
    void OnDrawGizmos()
    {
        Gizmos.DrawSphere(transform.position, _radius);
        Gizmos.DrawRay(transform.position + Vector3.up * 0.9f, transform.forward);
    }

    

    void JumpDelayTimer()
    {
        
        if (_jumpTimer <= 0.0f)
        {
            _canJump = true;
            _animator.SetBool(_animJump, false);
        }

        if (_jumpTimer > 0.0f)
        {
            _jumpTimer -= Time.deltaTime;
            _canJump = false;
        }
    }

    void Move()
    {
        
        float targetSpeed = _isSprint ? _sprintSpeed : _speed;
        float fallSpeed = _rigidbody.velocity.y;
        if (_direction.sqrMagnitude != 0)
        {
            Vector3 lookForward = new Vector3(_camTrans.forward.x, 0f, _camTrans.forward.z).normalized;
            Vector3 lookRight = new Vector3(_camTrans.right.x, 0f, _camTrans.right.z).normalized;
            //_moveDir = transform.TransformDirection(_direction).normalized * (targetSpeed * Time.deltaTime);
            _moveDir = (_direction.z * lookForward + _direction.x * lookRight) * (targetSpeed * Time.deltaTime);
            // 방향 일치시켜주기 - y를 안 넣는 이유는 앞으로 기울어지기 때문
            // transform.forward = lookForward로 하면 카메라에 따라 캐릭터도 회전하게 됨
            _playerBody.forward = _moveDir;
            _moveDir.y = fallSpeed;
            _rigidbody.velocity = _moveDir;
        }

        // Vector3 heading = _camera.localRotation * _direction;
        // transform.rotation = Quaternion.LookRotation(_direction);

        MoveAnimation(targetSpeed);
    }

    void MoveAnimation(float targetSpeed)
    {
        // 애니메이션
        float animSpeedY = _direction.z * targetSpeed / _moveAnimationMod;
        float animSpeedX = _direction.x * _speed / _moveAnimationMod;
        _animator.SetFloat(_animYSpeed, animSpeedY);
        _animator.SetFloat(_animXSpeed, animSpeedX);
    }

    void CheckGround()
    {
        if (Physics.CheckSphere(transform.position, _radius, 1 << 6, QueryTriggerInteraction.Ignore))
        {
            _isGrounded = true;
        }
        else
        {
            _isGrounded = false;
        }
    }

    public void AttackEvent(int damage)
    {
        
        RaycastHit hit;
        _animator.SetFloat(_animAttack, 0);
        Debug.DrawRay(transform.position + Vector3.up * 1.5f, transform.forward * 2);

        if (!Physics.Raycast(transform.position + Vector3.up * 0.9f, transform.forward, out hit, 1f))
            return;
        // TODO 디버그 삭제
        Debug.Log("Player Attack!");
        IDamageable damageable = hit.collider.GetComponent<IDamageable>();
        if (damageable.Equals(null))
            return;
        damageable.Damage(damage);
    }
}
```

<br><br>
#### PlayerRotation.cs
```C#
using System;
using System.Collections;
using System.Collections.Generic;
using Photon.Pun;
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerRotation : MonoBehaviourPunCallbacks
{
    [SerializeField] Transform _camTrans;
    [SerializeField] float deltaMod = 0.25f; 
    float _xRotation;
    float yRotation;
    PhotonView _pv;

    void Start()
    {
        Init();
    }

    void Init()
    {
        _pv = GetComponent<PhotonView>();
        if (_pv.IsMine && !_camTrans.gameObject.activeSelf)
            _camTrans.gameObject.SetActive(true);
    }
    
    // 쿼터뷰 카메라 ( LateUpdate : ) 
    void LateUpdate()
    {
        if (!_pv.IsMine)
            return;
        Follow();
    }

    // 쿼터뷰
    void Follow()
    {
        _camTrans.position = transform.position;
    }

    void OnLook(InputValue value)
    {
        Vector2 mouseDelta = value.Get<Vector2>();
        yRotation += mouseDelta.x * deltaMod;

        _xRotation -= mouseDelta.y;
        float xClampRotation = Mathf.Clamp(_xRotation * deltaMod, -30f, 40f);
        _camTrans.localRotation = Quaternion.Euler(xClampRotation, yRotation, 0f);
        
        /*Vector3 camAngle = camTrans.rotation.eulerAngles;
        float ClampAngleX = camAngle.x - mouseDelta.y;
        ClampAngleX = ClampAngleX < 180 ? Mathf.Clamp(ClampAngleX, 0f, 40f) : Mathf.Clamp(ClampAngleX, 330f, 360f);
        camTrans.rotation = Quaternion.Euler(ClampAngleX, camAngle.y + mouseDelta.x, camAngle.z);*/
    }
    
   
}

```


<br><br>
#### LivingEntity.cs
```C#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class LivingEntity : MonoBehaviour, IDamageable
{
    [SerializeField] protected int damage = 10;
    [SerializeField] protected int health = 10;
    public int Health { get => health; set => health = value; }

    public virtual void Damage(int amount)
    {
        health -= amount;
    }
    
     
}
```

<br><br>
#### Inventory.cs
```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System;

public enum ItemList
{
    Sword,
    Dagger,
    HpPotion
}

public class Inventory : MonoBehaviour
{
    public Dictionary<ItemList, GameObject> ItemDict { get; private set; } 
        = new Dictionary<ItemList, GameObject>();

   /* #region Singleton
    public static Inventory instance;

    private void Awake()
    {
        if(instance != null)
        {
            Destroy(gameObject);
            return;
        }
        instance = this;
    }
    #endregion*/

    public delegate void OnSlotCountChange(int val);
    public OnSlotCountChange onSlotCountChange;

    public delegate void OnChangeItem();
    public OnChangeItem onChangeItem;

    public List<Item> items = new List<Item>();
    int ItemLength;
    private int slotCnt;
    public int SlotCnt
    {
        get => slotCnt;
        set
        {
            slotCnt = value;
            onSlotCountChange.Invoke(slotCnt);
        }
    }


    // Start is called before the first frame update
    void Start()
    {
        Init();
        SlotCnt = 4;
    }

    

    void Init()
    {
        string[] itemNames = Enum.GetNames(typeof(ItemList));
        Array arr = Enum.GetValues(typeof(ItemList));
        
        for (int i = 0; i < arr.Length; i++)
        {
            ItemList item = (ItemList)arr.GetValue(i);
             ItemDict.Add(item, GetGameObejct(itemNames[i]));
             ItemDict[item].SetActive(false);
        }
    }

    GameObject GetGameObejct(string name)
    {
        foreach (Transform trans in transform.GetComponentsInChildren<Transform>())
        {
            if (trans.name == name)
                return trans.gameObject;
        }
        return null;
    }

    public bool AddItem(Item _item)
    {
        if(items.Count < SlotCnt)
        {
            items.Add(_item);
            if(onChangeItem != null)
            onChangeItem.Invoke();
            return true;
        }
        else
        {
            return false;
        }
    }

    public void RemoveItem(int _index)
    {
        items.RemoveAt(_index);
        onChangeItem.Invoke();
    }

    private void OnTriggerEnter(Collider collision)
    {
        if (collision.CompareTag("FieldItem"))
        {
           FieldItems fieldItems = collision.GetComponent<FieldItems>();
            if (AddItem(fieldItems.GetItem()))
            {
                fieldItems.DestroyItem();
            }
        }
    }

}

```


<br><br>
#### NetworkManager.cs
```C#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Photon.Pun;
using Photon.Realtime;
using TMPro;
using UnityEngine.SceneManagement;
using Random = UnityEngine.Random;


public class NetworkManager : MonoBehaviourPunCallbacks
{
    [SerializeField] TextMeshProUGUI inputText;
    static NetworkManager _instance;

    public static NetworkManager Instance
    {
        get
        {
            if (!_instance)
            {
                GameObject go = new GameObject("@NetworkManager");
                _instance = go.AddComponent<NetworkManager>();
            }
            
            return _instance;
        }
    }

    void Awake()
    {
        DontDestroyOnLoad(this);
    }

    public void ConnectBtn()
    {
       // PhotonNetwork.IsMessageQueueRunning = false;
        SceneManager.LoadScene(1);
        PhotonNetwork.ConnectUsingSettings();
        PhotonNetwork.NickName = inputText.text;
        
    }

    public override void OnConnectedToMaster()
    {
        PhotonNetwork.JoinOrCreateRoom("Room", new RoomOptions(), TypedLobby.Default);
        var player = PhotonNetwork.LocalPlayer;
        Debug.Log(player);
    }

    public override void OnJoinedRoom()
    {
        //PhotonNetwork.IsMessageQueueRunning = true;
        var pos = new Vector3(Random.Range(40f, 43f), 6f, Random.Range(66f, 70f));
        PhotonNetwork.Instantiate("Player", pos, Quaternion.identity);

    }

    [ContextMenu("PlayerList")]
    public void PlayerListPrint()
    {
        var players = PhotonNetwork.PlayerList;
        foreach (var VARIABLE in players)
        {
            Debug.Log(VARIABLE); 
        }
    }
}

```
<br><br>
#### NetworkManager.cs
```C#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Photon.Pun;
using Photon.Realtime;
using TMPro;
using UnityEngine.SceneManagement;
using Random = UnityEngine.Random;


public class NetworkManager : MonoBehaviourPunCallbacks
{
    [SerializeField] TextMeshProUGUI inputText;
    static NetworkManager _instance;

    public static NetworkManager Instance
    {
        get
        {
            if (!_instance)
            {
                GameObject go = new GameObject("@NetworkManager");
                _instance = go.AddComponent<NetworkManager>();
            }
            
            return _instance;
        }
    }

    void Awake()
    {
        DontDestroyOnLoad(this);
    }

    //포톤 접속 : 넘어갈 값 : 닉네임 
    public void ConnectBtn()
    {
       // PhotonNetwork.IsMessageQueueRunning = false;
        SceneManager.LoadScene(1);
        PhotonNetwork.ConnectUsingSettings();
        PhotonNetwork.NickName = inputText.text;
        
    }

    //마스터 클라이언트 체크 
    public override void OnConnectedToMaster()
    {
        PhotonNetwork.JoinOrCreateRoom("Room", new RoomOptions(), TypedLobby.Default);
        var player = PhotonNetwork.LocalPlayer;
        Debug.Log(player);
    }

    public override void OnJoinedRoom()
    {
        //PhotonNetwork.IsMessageQueueRunning = true;
        var pos = new Vector3(Random.Range(40f, 43f), 6f, Random.Range(66f, 70f));
        PhotonNetwork.Instantiate("Player", pos, Quaternion.identity);

    }

    [ContextMenu("PlayerList")]
    public void PlayerListPrint()
    {
        var players = PhotonNetwork.PlayerList;
        foreach (var VARIABLE in players)
        {
            Debug.Log(VARIABLE); 
        }
    }
}

```

<br><br>


![image](https://user-images.githubusercontent.com/74412438/199643688-cf86c11e-036e-4c80-b3c7-be6141cea6a8.png)
